- Feature Name: nano_contract_simulator
- Start Date: 2026-02-02
- Author: Andre Carneiro

# Summary
[summary]: #summary

An in-memory, self-contained simulator for NanoContract blueprints that lets developers write, test, and iterate on nano contracts without running a full Hathor node. The simulator faithfully reproduces the blueprint execution environment — including storage, actions, syscalls, call stacks, and reentrancy rules — while providing a high-level Python API that makes test authoring straightforward.

# Motivation
[motivation]: #motivation

Blueprint developers today must deploy contracts to a running node (or a local testnet) to validate their logic. This creates a slow feedback loop: start a node, fund wallets, submit transactions, wait for confirmations, inspect state through the node API. Unit-testing a single method can take minutes instead of milliseconds.

The simulator addresses this by:

- **Enabling fast iteration.** Tests execute entirely in-process with no I/O, making them suitable for `pytest` and CI pipelines.
- **Providing deterministic execution.** Seeds control RNG and ID generation, so the same test always produces the same result.
- **Exposing state directly.** Developers can inspect balances, storage fields, and contract existence directly from the simulator instance.
- **Supporting snapshot/restore.** Complex test scenarios can branch from a known state without re-executing setup logic.
- **Lowering the barrier to entry.** New developers can experiment with blueprints using only `hathorlib`, without understanding the full node architecture.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Core concepts

The simulator introduces four concepts a blueprint developer works with:

1. **Simulator** — the entry point. It manages blueprints, contracts, and provides the testing API.
2. **Addresses** — Can be base58 known addresses or a deterministic test identity created from human-readable names.
3. **Actions** — deposits, withdrawals, and authority grants/acquires that accompany method calls.
4. **Snapshots** — frozen copies of all simulator state that can be restored later.

## Writing a blueprint

Blueprints are regular Python classes that extend `Blueprint`.
State fields are declared as class-level annotations and methods that can be called by users are decorated with `@public` (state-mutating) or `@view` (read-only):

```python
from hathorlib.nanocontracts import Blueprint, public, view, Context

class SimpleCounter(Blueprint):
    count: int

    @public
    def initialize(self, ctx: Context) -> None:
        self.count = 0

    @public
    def increment(self, ctx: Context) -> None:
        self.count += 1

    @view
    def get_count(self) -> int:
        return self.count
```

Every blueprint must have an `initialize` method decorated with `@public`. This method is called exactly once when the contract is created.

## Testing a blueprint

```python
from hathorlib.simulator import Simulator

sim = Simulator()
sim.register_blueprint(SimpleCounter)

alice = sim.create_address("alice")
contract_id = sim.create_contract(SimpleCounter, caller=alice)

sim.call_public(contract_id, "increment", caller=alice)
count = sim.call_view(contract_id, "get_count")
assert count == 1
```

## Working with tokens and actions

When a method needs to move tokens, the caller attaches actions:

```python
from hathorlib.nanocontracts import NCDepositAction, TokenUid, HTR_TOKEN_UID as HTR

contract_id = sim.create_contract(
    RewardsProgramBlueprint,
    caller=alice,
    actions=[sim.deposit(HTR, 1000)],
)

sim.call_public(
    contract_id,
    "withdraw_rewards",
    caller=alice,
    actions=[sim.withdrawal(HTR, 50)],
)
```

The simulator's `deposit()`, `withdrawal()`, `grant_authority()` and `acquire_authority()` helper methods create action objects without requiring manual imports.

## Using the builder

For tests that share a common setup, `SimulatorBuilder` provides a fluent interface:

```python
from hathorlib.simulator import SimulatorBuilder

sim = (
    SimulatorBuilder()
    .register_blueprint(MyToken)
    .register_blueprint(MyMarket)
    .with_seed(b"test_seed")
    .with_timestamp(1700000000)
    .build()
)
```

## Snapshot and restore

```python
snapshot = sim.snapshot()

# Mutate state
sim.call_public(contract_id, "increment", caller=alice)
assert sim.call_view(contract_id, "get_count") == 2

# Roll back
sim.restore(snapshot)
assert sim.call_view(contract_id, "get_count") == 1
```

Snapshots deep-copy all storage, so restoring is safe regardless of what happened after the snapshot was taken.

## Inspecting state

```python
balance = sim.get_balance(contract_id)           # HTR balance
balance = sim.get_balance(contract_id, my_token)  # Custom token balance
all_balances = sim.get_all_balances(contract_id)
exists = sim.contract_exists(contract_id)
contracts = sim.get_all_contracts()
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Architecture

The simulator is organized into three layers:

- Layer 1: High-level API
    - Simulator, SimulatorBuilder
- Layer 2: Execution engine
    - SimulatorRunner
- Layer 3: In-memory storage
    - SimNcBlockStorage, SimNCContractStorage, SimNCChangesTracker


### Layer 1: Simulator (public API)

`Simulator` is the only class most devs will interact with. It owns:

| Field | Type | Purpose |
|-------|------|---------|
| `_block_storage` | `SimNCBlockStorage` | Root storage for all contracts |
| `_blueprints` | `dict[bytes, type[Blueprint]]` | Blueprint Class mapping |
| `_blueprint_ids` | `dict[type[Blueprint], BlueprintId]` | Reverse mapping |
| `_runner` | `SimulatorRunner \| None` | Lazily created execution engine |
| `_seed` | `bytes \| None` | RNG seed for determinism |
| `_timestamp` | `int` | Simulation timestamp |
| `_tx_counter` | `int` | Monotonic counter for unique IDs |
| `_address_cache` | `dict[str, Address]` | Name → Address cache |

The runner is lazily initialized and invalidated when a new blueprint is registered, so the blueprint registry is always in sync.

### Layer 2: SimulatorRunner (execution engine)

The runner is responsible for:

1. **Blueprint instantiation.** It creates a `Blueprint` instance backed by a `BlueprintEnvironment` that delegates syscalls back to the runner.
2. **Call stack management.** A `CallInfo` object tracks the current execution stack, enforcing `MAX_RECURSION_DEPTH` (100) and `MAX_CALL_COUNTER` (250).
3. **Action execution.** Before a method runs, the runner applies deposit/withdrawal/authority actions to the contract's changes tracker.
4. **Reentrancy enforcement.** By default, calling the same contract while it is already on the call stack raises an error. Methods can opt in to reentrancy with `@public(allow_reentrancy=True)`.
5. **View method protection.** After a `@view` method returns, the runner verifies no storage changes were made. Any mutation raises `NCViewMethodError`.
6. **Balance validation.** After a `@public` method returns, all token balances are checked to be non-negative.
7. **Field initialization validation.** After `initialize()`, the runner checks that all declared fields have been assigned.

#### Syscalls

The runner implements the full syscall interface that `BlueprintEnvironment` exposes to blueprint code:

| Syscall | Effect |
|---------|--------|
| `get_rng()` | Returns a per-contract deterministic `NanoRNG` |
| `revoke_authorities()` | Removes mint/melt flags from a token balance |
| `mint_tokens()` | Increases token supply (requires mint authority) |
| `melt_tokens()` | Decreases token supply (requires melt authority) |
| `emit_event()` | Records a custom event (no-op in storage) |
| `change_blueprint()` | Swaps the contract's blueprint ID |
| `create_child_deposit_token()` | Creates a new deposit-backed token |
| `create_child_fee_token()` | Creates a new fee-based token |
| `call_another_contract_view_method()` | Executes a view method on a different contract |
| `proxy_call_view_method()` | Calls a view method using another blueprint's code |
| `create_another_contract()` | Creates a child contract from within a method |

### Layer 3: Storage

#### SimNCBlockStorage

The top-level storage container. It holds:

- `_contracts: dict[bytes, SimNCContractStorage]` — all contract storages keyed by contract ID.
- `_address_seqnums: dict[bytes, int]` — per-address sequence numbers.
- `_root_ids: dict[bytes, bytes]` — contract ID to root contract ID mapping.

`copy()` performs a deep copy of the entire block storage, which is the mechanism behind `Simulator.snapshot()`.

#### SimNCContractStorage

Per-contract state. It stores:

- `_data: dict[bytes, bytes]` — serialized key-value pairs for blueprint fields.
- `_balances: dict[bytes, MutableBalance]` — token balances with mint/melt authority flags.
- `_blueprint_id: BlueprintId | None` — the blueprint this contract was created from.
- `_locked: bool` — when locked, all mutations raise an error.

The storage exposes typed access through `get_obj(key, nc_type, default)` and `put_obj(key, nc_type, obj)`, which handle serialization/deserialization using the NanoContract type system.

#### SimNCChangesTracker

A write-ahead layer that wraps a `SimNCContractStorage`. During method execution, all reads go through the tracker (checking local changes first, then falling through to the underlying storage) and all writes are buffered locally.

Key behaviors:

- **Lazy commit.** Changes are only flushed to the underlying storage when `commit()` is called.
- **Rollback.** Calling `block()` instead of `commit()` discards all buffered changes.
- **Balance validation.** `validate_balances_are_positive()` checks that no token balance went negative, combining local deltas with the underlying storage values.
- **Emptiness check.** `is_empty()` returns `True` if no modifications were made — used by the runner to enforce view method did not mutate storage.

## Deterministic ID generation

All identifiers are derived from SHA-256, ensuring reproducibility:

| ID type | Input |
|---------|-------|
| Blueprint ID | `SHA256(class_name)` |
| Contract ID | `SHA256("contract:{counter}:{timestamp}")` |
| Transaction hash | `SHA256("tx:{counter}:{timestamp}")` |
| Address | P2PKH envelope around `SHA256("test_address:{name}")[:20]` |

The `_tx_counter` is a monotonically increasing integer shared between contract ID and transaction hash generation, guaranteeing uniqueness within a simulation run.

## Context creation

Each method call receives a `Context` containing:

- `caller_id` — the `Address` or `ContractId` of the caller.
- `vertex` — a `VertexData` with the transaction hash (other fields use simulation defaults).
- `block` — a `BlockData` with the simulation timestamp.
- `actions` — a dictionary grouping `NCAction` objects by `TokenUid`.

The context is immutable. The runner passes `ctx.copy()` to the blueprint method so that blueprint code cannot corrupt the runner's copy.

## Action types

| Action | Fields | Purpose |
|--------|--------|---------|
| `NCDepositAction` | `_token_uid`, `amount` | Caller sends tokens to contract |
| `NCWithdrawalAction` | `_token_uid`, `amount` | Contract sends tokens to caller |
| `NCGrantAuthorityAction` | `_token_uid`, `mint`, `melt` | Caller grants authority to contract |
| `NCAcquireAuthorityAction` | `_token_uid`, `mint`, `melt` | Caller takes authority back |

Methods must explicitly declare which action types they accept via decorator parameters (`allow_deposit`, `allow_withdrawal`, `allow_grant_authority`, `allow_acquire_authority`). The default is to forbid all actions (whitelist approach).

# Drawbacks
[drawbacks]: #drawbacks

- **Fidelity gap.** The simulator does not model transaction DAG structure, consensus, or network propagation. A blueprint that passes simulator tests may still fail in production if it depends on vertex ordering, confirmation times, or full transaction validation.
- **Maintenance burden.** The simulator must stay in sync with the real execution engine. Changes to the runner, syscall interface, or storage semantics in `hathor-core` must be mirrored in the simulator, or tests will give false confidence.
- **No fuel metering.** The simulator does not enforce execution fuel limits. A blueprint that runs fine in the simulator could exceed fuel limits on-chain.
- **Simplified token model.** Token creation in the simulator generates synthetic token UIDs and does not enforce token rules.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why in-memory storage instead of SQLite or RocksDB?

In-memory `dict`-based storage was chosen because:

- It requires zero external dependencies.
- It makes snapshot/restore trivial (`copy.deepcopy` semantics via `copy()` methods).
- It keeps test execution fast — no filesystem I/O or serialization overhead.
- It is sufficient for the expected data sizes in tests (dozens, not thousands, of contracts).

SQLite was considered but rejected: it would add complexity to snapshots (requiring WAL checkpointing or file copies) and slow down the common case where state fits easily in memory.

## Why deterministic IDs instead of random UUIDs?

Deterministic IDs make tests reproducible without requiring snapshot files or fixtures. Two runs of the same test produce identical contract IDs, addresses, and transaction hashes. This simplifies debugging and allows assertion on specific ID values when needed.

# Prior art
[prior-art]: #prior-art

- **Ethereum's Hardhat / Foundry.** Both provide in-process EVM execution for testing Solidity contracts. Hardhat uses a JavaScript EVM implementation; Foundry uses a Rust EVM. The hathorlib simulator follows the same principle — bring the execution environment into the test process — but differs in that it operates on Python classes rather than compiled bytecode.
- **Solana's `solana-program-test`.** Provides a `BanksClient` that runs a stripped-down Solana runtime in-process. Similar to our approach of replacing only the storage and network layers while keeping the execution logic intact.
- **NEAR's `near-workspaces`.** Offers both sandbox (local node) and simulation (in-process) modes. Our simulator is closest to their simulation mode, trading full fidelity for speed.
- **Tezak's `tezos-client mockup` mode.** Runs a local, single-node Tezos environment without networking. More heavyweight than our approach but offers higher fidelity.

The common lesson from all of these is that fast, in-process testing is essential for developer productivity, even if it comes at the cost of not simulating every aspect of the production environment. Our simulator follows this established pattern.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Fuel metering.** Should the simulator enforce execution fuel limits to catch contracts that would exceed on-chain limits? This could be added later without changing the API, but the constants and metering logic would need to be extracted from the full node.
- **Time advancement.** The simulator currently uses a fixed timestamp. Should it provide a `sim.advance_time(seconds)` method for testing time-dependent logic?
- **Event inspection.** Events emitted via `syscall.emit_event()` are currently not captured for test assertions. Should the simulator collect them and expose a `sim.get_events(contract_id)` method?
- **Error reporting.** When a method call fails, the error message includes the Python traceback but not the simulated call stack. Should the simulator produce richer error diagnostics that show the full contract call chain?

# Future possibilities
[future-possibilities]: #future-possibilities

- **Integration with fullnode simulator.** We could use the fullnode simulator to create simulations that test actual network scenarios, not only blueprint execution.
- **Snapshot serialization.** Snapshots could be serialized to disk, enabling test scenarios that start from a pre-built state without re-executing setup logic.
