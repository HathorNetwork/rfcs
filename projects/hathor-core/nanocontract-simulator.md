- Feature Name: nano_contract_simulator
- Start Date: 2026-02-02
- Author: Andre Carneiro

# Summary
[summary]: #summary

An in-memory, self-contained simulator for NanoContract blueprints that lets developers write, test, and iterate on nano contracts without running a full Hathor node. The simulator uses the same blueprint environment that the full node uses — all of which live in `hathorlib`. This gives near-perfect fidelity with zero maintenance divergence, while providing a high-level Python API that makes test authoring straightforward.

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
from hathorlib.nanocontracts import HTR_TOKEN_UID as HTR

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

The blueprint execution infrastructure (Runner, storage classes, field system, type system) lives in `hathorlib`. The simulator reuses these production components directly, substituting only two things:

1. **Storage backend**: `InMemoryNodeTrieStore` instead of `RocksDBNodeTrieStore`.
2. **Transaction data source**: `SimulatorTransactionStorage` (implements `TransactionStorageProtocol`) instead of the full node's `TransactionStorage`.

The simulator is organized into three layers:

- Layer 1: High-level API (simulator-specific)
    - `Simulator`, `SimulatorBuilder`
- Layer 2: Execution engine (reused from `hathorlib`)
    - `Runner`, `CallInfo`, `BlueprintEnvironment`, `BalanceRules`, `MeteredExecutor`
- Layer 3: Storage (reused from `hathorlib`, with in-memory backend)
    - `NCBlockStorage`, `NCContractStorage`, `NCChangesTracker`, `PatriciaTrie`, `InMemoryNodeTrieStore`

### Layer 1: Simulator (public API)

`Simulator` is the only class most devs will interact with. It owns:

| Field            | Type                          | Purpose                                                                              |
| ---------------- | ----------------------------- | ------------------------------------------------------------------------------------ |
| `_block_storage` | `NCBlockStorage`              | Root storage for all contracts (production class, backed by `InMemoryNodeTrieStore`) |
| `_tx_storage`    | `SimulatorTransactionStorage` | Implements `TransactionStorageProtocol` for blueprint/token lookups                  |
| `_runner`        | `Runner \= None`              | Lazily created                                                                       |
| `_seed`          | `bytes \= None`               | RNG seed for determinism                                                             |
| `_timestamp`     | `int`                         | Simulation timestamp                                                                 |
| `_tx_counter`    | `int`                         | Monotonic counter for unique IDs                                                     |
| `_address_cache` | `dict[str, Address]`          | Name to Address cache                                                                |

The runner is lazily initialized and invalidated when a new blueprint is registered, so the blueprint registry is always in sync.

### Layer 2: Runner (reused execution engine)

The simulator uses the same `Runner` class from `hathorlib.nanocontracts.runner` that the network uses. No simulator-specific runner is needed. The Runner is responsible for:

1. **Blueprint instantiation.** It creates a `Blueprint` instance backed by a `BlueprintEnvironment` that delegates syscalls back to the runner.
2. **Call stack management.** A `CallInfo` object tracks the current execution stack, enforcing `MAX_RECURSION_DEPTH` (100) and `MAX_CALL_COUNTER` (250).
3. **Action execution.** Before a method runs, the runner applies deposit/withdrawal/authority actions to the contract's changes tracker.
4. **Reentrancy enforcement.** By default, calling the same contract while it is already on the call stack raises an error. Methods can opt in to reentrancy with `@public(allow_reentrancy=True)`.
5. **View method protection.** After a `@view` method returns, the runner verifies no storage changes were made. Any mutation raises `NCViewMethodError`.
6. **Balance validation.** After a `@public` method returns, all token balances are checked to be non-negative.
7. **Field initialization validation.** After `initialize()`, the runner checks that all declared fields have been assigned.

The Runner accesses external data (blueprint classes, token information) through a `TransactionStorageProtocol`, which the Simulator satisfies with a `SimulatorTransactionStorage` — an in-memory registry that maps blueprint IDs to classes and provides synthetic token metadata.

### Layer 3: Storage (reused with in-memory backend)

The simulator uses the same storage classes from `hathorlib.nanocontracts.storage` that the network uses. The only difference is the lowest-level storage backend: `InMemoryNodeTrieStore` instead of `RocksDBNodeTrieStore`.

#### NodeTrieStore Protocol and InMemoryNodeTrieStore

`NodeTrieStore` is a `typing.Protocol` that defines the interface for the lowest-level key-value store used by `PatriciaTrie`.
The hathorlib provides an `InMemoryNodeTrieStore`, a simple implementation that saves state on a `dict[bytes, Node]` which makes managing the state simple and copying the state easier (e.g. when creating a snapshot).

On the fullnode, `RocksDBNodeTrieStore` implements the same protocol backed by RocksDB. Since `PatriciaTrie` only uses these four dunder methods, the protocol is sufficient to swap backends transparently.

#### TransactionStorageProtocol and SimulatorTransactionStorage

`TransactionStorageProtocol` defines the interface the Runner uses to look up blueprint classes and token information:

```python
class TransactionStorageProtocol(Protocol):
    def get_blueprint_class(self, blueprint_id: BlueprintId) -> type[Blueprint]: ...
    def get_token_creation_transaction(self, token_uid: TokenUid) -> Any: ...
```

The simulator provides `SimulatorTransactionStorage`, which resolves blueprint classes from an in-memory registry and returns synthetic token metadata.
On the network, the full node's `TransactionStorage` already satisfies this protocol.

The `TransactionStorageProtocol` will hold:

- `_blueprints: dict[bytes, <blueprint class>]` — all blueprints created by the user.
  - These blueprints simulate previously created blueprints via OnChainBlueprint transactions.
- `_tokens: dict[bytes, TokenInfo]` — all user generated tokens keyed by token UID.
  - These tokens simulate the usual custom tokens on the network.


#### NCBlockStorage

The production top-level storage container, used as-is. It holds the storage used in the contracts and other data required for the nanocontracts to work.
This block storage is saved per-block and can be copied if we copy the underlying `PatriciaTrie` which holds the state, this is the mechanism behind `Simulator.snapshot()`

#### NCContractStorage

The production per-contract state container, used as-is. Each contract's `PatriciaTrie` is backed by an `InMemoryNodeTrieStore` in the simulator (vs. `RocksDBNodeTrieStore` on the network). It stores:

- Serialized key-value pairs for blueprint fields (via `PatriciaTrie`).
- Token balances with mint/melt authority flags.
- The blueprint ID this contract was created from.

The storage exposes typed access through `get_obj(key, nc_type, default)` and `put_obj(key, nc_type, obj)`, which handle serialization/deserialization using the NanoContract type system.

#### NCChangesTracker

The production write-ahead layer, used as-is. It wraps an `NCContractStorage`. During method execution, all reads go through the tracker (checking local changes first, then falling through to the underlying storage) and all writes are buffered locally.

Key behaviors:

- Lazy commit.
  - Changes are only flushed to the underlying storage when `commit()` is called.
- Rollback.
  - Calling `block()` instead of `commit()` discards all buffered changes.
  - This makes a failed execution not affect the underlying storage.
- Balance validation.
  - `validate_balances_are_positive()` checks that no token balance went negative, combining local deltas with the underlying storage values.
- Emptiness check.
  - `is_empty()` returns `True` if no modifications were made — used by the runner to enforce view method did not mutate storage.

## Deterministic ID generation

All identifiers are derived from SHA-256, ensuring reproducibility:

| ID type          | Input                                                      |
| ---------------- | ---------------------------------------------------------- |
| Blueprint ID     | `SHA256(class_name)`                                       |
| Contract ID      | `SHA256("contract:{counter}:{timestamp}")`                 |
| Transaction hash | `SHA256("tx:{counter}:{timestamp}")`                       |
| Address          | P2PKH envelope around `SHA256("test_address:{name}")[:20]` |

The `_tx_counter` is a monotonically increasing integer shared between contract ID and transaction hash generation, guaranteeing uniqueness within a simulation run.

## Context creation

Each method call receives a `Context` containing:

- `caller_id` — the `Address` or `ContractId` of the caller.
- `vertex` — a `VertexData` with the transaction hash (other fields use simulation defaults).
- `block` — a `BlockData` with the simulation timestamp.
- `actions` — a dictionary grouping `NCAction` objects by `TokenUid`.

The context is immutable. The runner passes `ctx.copy()` to the blueprint method so that blueprint code cannot corrupt the runner's copy.

## Action types

| Action                     | Fields                       | Purpose                             |
| -------------------------- | ---------------------------- | ----------------------------------- |
| `NCDepositAction`          | `_token_uid`, `amount`       | Caller sends tokens to contract     |
| `NCWithdrawalAction`       | `_token_uid`, `amount`       | Contract sends tokens to caller     |
| `NCGrantAuthorityAction`   | `_token_uid`, `mint`, `melt` | Caller grants authority to contract |
| `NCAcquireAuthorityAction` | `_token_uid`, `mint`, `melt` | Caller takes authority back         |

Methods must explicitly declare which action types they accept via decorator parameters (`allow_deposit`, `allow_withdrawal`, `allow_grant_authority`, `allow_acquire_authority`). The default is to forbid all actions (whitelist approach).

# Drawbacks
[drawbacks]: #drawbacks

### Simplified token model

Token creation in the simulator generates synthetic token UIDs and does not enforce the full token creation rules (e.g., deposit amounts, token naming validation beyond what the Runner checks).

### Prerequisite Refactoring

The simulator depends on the nanocontract infrastructure being available in `hathorlib`. This refactoring must be completed first, adding complexity to the overall project timeline.
After the refactoring:

- All blueprint definition and execution code lives in `hathorlib`.
- Only on-chain-specific code stays in `hathor-core`: `BlockExecutor`, `RocksDBNodeTrieStore`, and transaction sorters.
- The `Runner` uses a new `TransactionStorageProtocol` (a `typing.Protocol`) to access blueprint classes and token information, decoupling it from the full node's `TransactionStorage`.
- `NodeTrieStore` (the lowest-level storage backend used by `PatriciaTrie`) becomes a `typing.Protocol`, with `RocksDBNodeTrieStore` in `hathor-core` and a new `InMemoryNodeTrieStore` in `hathorlib`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why reuse the production Runner instead of a custom SimulatorRunner?

The previous design used a dedicated `SimulatorRunner` that reimplemented the execution engine. The current design reuses the production `Runner` from `hathorlib` directly. This was made possible by decoupling the Runner from the full node via `TransactionStorageProtocol` and `NodeTrieStore` Protocol.

Benefits:
- **Near-zero fidelity gap.** The simulator runs blueprints with the exact same Runner, syscall implementations, balance validation, reentrancy checks, call stack management, and field initialization logic as the network. The only difference is the storage backend.
- **Near-zero maintenance burden.** There is no duplicate execution logic. When the Runner is updated in `hathorlib`, the simulator automatically gets the same behavior.
- **Fuel metering included.** Since the production `MeteredExecutor` is available in `hathorlib`, the simulator can enforce the same fuel and memory limits as on-chain execution.

## Why InMemoryNodeTrieStore instead of SQLite or RocksDB?

The `InMemoryNodeTrieStore` is a simple `dict`-backed implementation of the `NodeTrieStore` protocol. It was chosen because:

- It requires zero external dependencies.
- It makes snapshot/restore trivial (`copy.deepcopy` semantics via `copy()` methods).
- It keeps test execution fast — no filesystem I/O or serialization overhead.
- It is sufficient for the expected data sizes in tests (dozens, not thousands, of contracts).
- It plugs into the existing `NCBlockStorage` / `NCContractStorage` / `PatriciaTrie` stack transparently via the `NodeTrieStore` protocol.

SQLite was considered but rejected: it would add complexity to snapshots (requiring WAL checkpointing or file copies) and slow down the common case where state fits easily in memory.

## Why deterministic IDs instead of random UUIDs?

Deterministic IDs make tests reproducible without requiring snapshot files or fixtures. Two runs of the same test produce identical contract IDs, addresses, and transaction hashes. This simplifies debugging and allows assertion on specific ID values when needed.

# Prior art
[prior-art]: #prior-art

- **Ethereum's Hardhat / Foundry.** Both provide in-process EVM execution for testing Solidity contracts. Hardhat uses a JavaScript EVM implementation; Foundry uses a Rust EVM. The hathorlib simulator follows the same principle — bring the execution environment into the test process — but differs in that it operates on Python classes rather than compiled bytecode.
- **Solana's `solana-program-test`.** Provides a `BanksClient` that runs a stripped-down Solana runtime in-process. Similar to our approach of replacing only the storage backend and transaction data source while keeping the entire execution engine intact.
- **NEAR's `near-workspaces`.** Offers both sandbox (local node) and simulation (in-process) modes. Our simulator is closest to their simulation mode, trading full fidelity for speed.
- **Tezak's `tezos-client mockup` mode.** Runs a local, single-node Tezos environment without networking. More heavyweight than our approach but offers higher fidelity.

The common lesson from all of these is that fast, in-process testing is essential for developer productivity, even if it comes at the cost of not simulating every aspect of the production environment. Our simulator follows this established pattern, and goes further than most by reusing the production execution engine rather than reimplementing it.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Fuel metering opt-in.** The production `MeteredExecutor` is available in `hathorlib` and can be used by the simulator. Should fuel metering be enabled by default, or should it be opt-in (e.g., `Simulator(enable_fuel_metering=True)`)? Default-on gives maximum fidelity; default-off gives a simpler experience for new developers.
- **Time advancement.** The simulator currently uses a fixed timestamp. Should it provide a `sim.advance_time(seconds)` method for testing time-dependent logic?
- **Event inspection.** Events emitted via `syscall.emit_event()` are currently not captured for test assertions. Should the simulator collect them and expose a `sim.get_events(contract_id)` method?
- **Error reporting.** When a method call fails, the error message includes the Python traceback but not the simulated call stack. Should the simulator produce richer error diagnostics that show the full contract call chain?

# Future possibilities
[future-possibilities]: #future-possibilities

- **Integration with fullnode simulator.** We could use the fullnode simulator to create simulations that test actual network scenarios, not only blueprint execution.
- **Snapshot serialization.** Snapshots could be serialized to disk, enabling test scenarios that start from a pre-built state without re-executing setup logic.
