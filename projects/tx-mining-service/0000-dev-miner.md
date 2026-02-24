# Simplified Mining for Integration Tests

- Feature Name: `dev-miner-integration-tests`
- Start Date: 2026-02-23
- Author: Tulio (tulio@hathor.network)

## Summary

The current integration test infrastructure for `hathor-wallet-lib` runs a private blockchain with a full stratum-based mining stack (fullnode + tx-mining-service + cpuminer), identical to what would be used in production with multiple nodes and miners. This creates unnecessary complexity, wasted processing power, and fluctuations in test results that require constant workarounds.

This RFC introduces a **dev-miner mode** across three repositories—[`hathor-core`](https://github.com/HathorNetwork/hathor-core), [`tx-mining-service`](https://github.com/HathorNetwork/tx-mining-service), and [`hathor-wallet-lib`](https://github.com/HathorNetwork/hathor-wallet-lib)—to replace the heavyweight mining pipeline with a lightweight, deterministic alternative tailored for integration tests.

## Motivation

Integration tests in `hathor-wallet-lib` exercise wallet operations (sending transactions, creating tokens, minting, melting, delegating authority, etc.) against a real private blockchain. These tests have historically suffered from several pain points:

1. **Stratum overhead**: The production mining path involves a fullnode producing block templates, `tx-mining-service` managing a stratum server, and `cpuminer` solving PoW via the stratum protocol. For a single-node, single-miner test scenario, this multi-process coordination is just overhead.

2. **Non-deterministic block timing**: `cpuminer` solves blocks as fast as it can, producing irregular block intervals. Some tests rely on block timing (e.g., waiting for confirmations, reward lock releases), and the variable pace forces test helpers to include generous timeouts and retry loops.

3. **Processing waste**: Even with `--test-mode-tx-weight` reducing transaction PoW to trivial difficulty, blocks still require the full Difficulty Adjustment Algorithm (DAA) weight. On CI machines with limited CPU, this means block mining competes with test execution for resources.

4. **Fragile fund injection**: The test suite used a genesis wallet to inject funds into test wallets. This approach required pre-calculated wallet addresses and careful UTXO management to avoid conflicts between parallel test files. When UTXOs were exhausted or timing was off, tests failed non-deterministically.

The goal is a testing infrastructure where:
- Block production is regular and predictable
- Transaction PoW is solved instantly (in-process, no stratum round-trip)
- Fund injection is reliable and conflict-free

## Guide-level explanation

### Before: Three processes, one miner

Previously, running integration tests spun up four Docker containers:

```
fullnode ──stratum──▶ tx-mining-service ◀──stratum──── cpuminer
                           ▲
                           │ HTTP
                     wallet-lib tests
```

The `cpuminer` connected to `tx-mining-service` via the stratum protocol, received block templates, solved PoW, and submitted solutions. When a wallet-lib test submitted a transaction via `tx-mining-service`'s HTTP API, the service queued it for the next available stratum miner (the same `cpuminer`), which had to pause block mining, solve the transaction's PoW, and return to block mining.

### After: One process, built-in mining

Now, the test composition uses three containers (no `cpuminer`):

```
fullnode ◀──HTTP──── tx-mining-service (dev-miner mode)
                           ▲
                           │ HTTP
                     wallet-lib tests
```

When `tx-mining-service` starts with `--dev-miner`:
- **Transactions** are mined synchronously in-process. The test submits a transaction, the service solves the trivial PoW directly (no stratum round-trip), and returns the result.
- **Blocks** are mined by a background loop that polls the fullnode for templates, solves them, and submits them at a configurable interval (`--block-interval`).

Combined with the new `--test-mode-block-weight` flag on the fullnode (which reduces block weight to 1, just like `--test-mode-tx-weight` does for transactions), both block and transaction PoW become trivially solvable—most nonces pass on the first try.

## Reference-level explanation

### Changes by repository

#### `hathor-core` (branch: `feature/test-mode-block-weight`)
Proof of Concept code branch: https://github.com/tuliomir/hathor-core/compare/master...tuliomir:hathor-core:feature/test-mode-block-weight

**One commit, three files changed.**

The existing `TestMode` enum in [`hathor/daa.py`](https://github.com/HathorNetwork/hathor-core/blob/master/hathor/daa.py) was an `IntFlag` with a single value (`TEST_TX_WEIGHT = 1`). This RFC adds `TEST_BLOCK_WEIGHT = 2` and `TEST_ALL_WEIGHT = 3` (both flags combined):

```python
class TestMode(IntFlag):
    DISABLED = 0
    TEST_TX_WEIGHT = 1
    TEST_BLOCK_WEIGHT = 2
    TEST_ALL_WEIGHT = 3
```

When `TEST_BLOCK_WEIGHT` is set, `calculate_block_difficulty()` and `calculate_next_weight()` return `1.0` unconditionally, bypassing the full DAA computation.

The CLI builder ([`hathor_cli/builder.py`](https://github.com/HathorNetwork/hathor-core/blob/master/hathor_cli/builder.py)) was updated to use bitwise OR (`|=`) when composing test modes, so both flags can be enabled simultaneously:

```python
if self._args.test_mode_tx_weight:
    test_mode |= TestMode.TEST_TX_WEIGHT
if self._args.test_mode_block_weight:
    test_mode |= TestMode.TEST_BLOCK_WEIGHT
```

A new `--test-mode-block-weight` argument was added to [`run_node.py`](https://github.com/HathorNetwork/hathor-core/blob/master/hathor_cli/run_node.py) and registered in the [unsafe arguments list](https://github.com/HathorNetwork/hathor-core/blob/master/hathor_cli/run_node.py#L50).

#### `tx-mining-service` (branch: `feature/devminer`)
Proof of Concept code branch: https://github.com/tuliomir/tx-mining-service/compare/master...tuliomir:tx-mining-service:feat/dev-miner

**Six commits, new `txstratum/dev/` package with three modules.**

The main addition is a `txstratum/dev/` package containing:

1. **`tx_miner.py`** — A `solve_tx()` function that iterates the nonce space to find a valid PoW solution for a transaction. With `--test-mode-tx-weight` on the fullnode, transaction weight is ~1, so this typically succeeds on the first nonce.

2. **`block_miner.py`** — A `BlockMiner` class that runs as an async background loop:
   - Waits for the fullnode to become available (by attempting `get_block_template`)
   - Fetches a block template, offloads PoW solving to a thread executor, and submits the solved block
   - Compensates for solving time to maintain a steady block interval: if solving takes `T` seconds and the interval is `I` seconds, it sleeps `max(0, I - T)` before the next cycle

3. **`manager.py`** — A `DevMiningManager` that implements the same interface as the production [`TxMiningManager`](https://github.com/HathorNetwork/tx-mining-service/blob/master/txstratum/manager.py) (used by the [HTTP API layer](https://github.com/HathorNetwork/tx-mining-service/blob/master/txstratum/api.py)), but mines transactions in-process instead of delegating to stratum miners. Key behaviors:
   - Transaction jobs are solved asynchronously via `run_in_executor` to avoid blocking the event loop
   - Parent resolution (`add_parents`) is handled inline
   - Job lifecycle (status tracking, cleanup scheduling, cancellation) mirrors the production manager
   - The `status()` endpoint includes a `dev_miner: true` flag

The CLI ([`txstratum/cli.py`](https://github.com/HathorNetwork/tx-mining-service/blob/master/txstratum/cli.py)) gains two new arguments:
- `--dev-miner`: Enables dev-miner mode, routing to `RunDevService` instead of [`RunService`](https://github.com/HathorNetwork/tx-mining-service/blob/master/txstratum/cli.py#L151)
- `--block-interval`: Block mining interval in milliseconds (default: 1000)

A comprehensive test suite (`tests/test_dev_miner.py`) covers:
- Transaction PoW solving at trivial and standard weights
- Full job lifecycle through the HTTP API (submit, poll, done)
- Parent resolution, propagation, health checks, mining status
- Block mining interval regularity with both trivial and simulated slow mining

#### `hathor-wallet-lib` (branch: `feat/parallel-integration`)
Proof-of-concept branch: https://github.com/tuliomir/hathor-wallet-lib/compare/master...tuliomir:hathor-wallet-lib:feat/parallel-integration <br/>
_Note: The contents of this branch are less relevant to this RFC because it only consumes the results produced here, confirming they work in a live test. However, it contains additional changes that refer to parallel tests, not in the scope of this RFC._

**Changes across 12 test files, 1 new helper module, Docker composition update, and setup changes.**

**Docker composition** ([`docker-compose.yml`](https://github.com/HathorNetwork/hathor-wallet-lib/blob/master/__tests__/integration/configuration/docker-compose.yml)):
- Fullnode image switched from a parameterized image to `hathor-core:test-mode-block-weight` (proposed build with the new flag)
- Added `--test-mode-block-weight` to the fullnode command
- Replaced `cpuminer` container with `--dev-miner` flag on `tx-mining-service`

**Setup changes** ([`setupTests-integration.js`](https://github.com/HathorNetwork/hathor-wallet-lib/blob/master/setupTests-integration.js)):
- [TX weight constants](https://github.com/HathorNetwork/hathor-wallet-lib/blob/master/src/constants.ts#L204) (`txMinWeight`, `txMinWeightK`, `txWeightCoefficient`) set to their minimum values to match the fullnode's privnet configuration, ensuring `calculateWeight()` produces minimal weights

### Interaction between components

The full flow for a test that creates a custom token:

1. Test calls `fundAddress(hWallet, address, 10n)` ( a custom change, pre-requisite to parallel tests. Mentioned here only to clarify the full flow )
2. `fundAddress()` sends `POST /fund { address, amount: 10 }` to the test helper service
3. Test helper service creates a transaction from its UTXO pool and submits it to `tx-mining-service`
4. `tx-mining-service` (in dev-miner mode) solves the trivial PoW in-process and pushes the transaction to the fullnode
5. The fullnode validates and accepts the transaction
6. `fundAddress()` waits for the wallet's WebSocket to receive the transaction notification
7. Test proceeds to `createTokenHelper()`, which internally calls `sendTransaction()`, which submits another transaction through the same `tx-mining-service` → fullnode path

Meanwhile, the `BlockMiner` in `tx-mining-service` is independently producing blocks at a steady 1-second cadence, advancing the blockchain so that reward locks release and confirmations accumulate.

## Drawbacks

**Divergence from production mining path**: By bypassing the stratum protocol entirely, dev-miner mode no longer exercises the code path that real mining uses. Bugs in the stratum implementation or in the miner↔service interaction would not be caught by integration tests.

Potential issues can be mitigated by adding a test with the production `tx-mining-service` only on rare and strategic moments, such as when there is a new tag for a release candidate. However, changes to the `tx-mining-service` code should preferably be testes by integration tests in its own repository, requesting execution from all clients pointing to the new code instead of the other way around.


## Rationale and alternatives

### Why `--test-mode-block-weight` as a fullnode flag?

An alternative would be to have `tx-mining-service` modify block templates to override their weight before solving. However, the fullnode validates blocks on submission and rejects those whose weight doesn't match the DAA output. The cleanest solution is to have the DAA itself return a trivial weight, which requires a fullnode flag. This mirrors the existing [`--test-mode-tx-weight`](https://github.com/HathorNetwork/hathor-core/blob/master/hathor_cli/run_node.py#L89) pattern, keeping the two mechanisms parallel and discoverable.

Additionally, the tests from the fullnode itself already use a feature that reduces the block weight limit. This feature is just being exposed for external use in proposed change.

### Why an in-process miner instead of a faster cpuminer?

We considered keeping `cpuminer` but with a lower difficulty target. The problem isn't just mining speed—it's the stratum coordination overhead and the non-deterministic timing. Even with trivial difficulty, `cpuminer` produces blocks as fast as it can (no interval control) with as much processing power as it can ( even with the limit of a single thread ), and the [stratum protocol](https://github.com/HathorNetwork/tx-mining-service/blob/master/txstratum/protocol.py) adds latency to transaction mining. The in-process miner gives us direct control over both block cadence and transaction PoW resolution.

### Create a new repository for the dev-miner

If we consider the change to the production `tx-mining-service` potentially dangerous or compromising, a new dedicated `tx-mining-service-dev` repository could be built, containing exclusively this simplified mining code.

### What if we do nothing?

The existing infrastructure works but continues to impose costs:
- CI runs are slower than necessary
- New test authors hit non-deterministic failures and must learn the workarounds
- The cpuminer container consumes CPU that could be used by the tests themselves, or slows down local user development environments

## Prior art

- **Bitcoin's regtest mode**: Bitcoin Core provides a regression test mode where blocks can be generated on demand via RPC (`generatetoaddress`), bypassing normal PoW requirements entirely. Our approach is similar in spirit but operates at the mining service layer rather than within the fullnode, preserving the fullnode's validation logic.

- **Ethereum's Hardhat/Ganache**: Ethereum test frameworks often include built-in block production (auto-mine on transaction, or interval mining) without external mining processes. The dev-miner mode provides comparable functionality for the Hathor test environment.

## Unresolved questions

- **Block interval tuning**: The default 1-second block interval works for current tests but may need adjustment as the test suite grows. Should this be configurable per-test-file or remain a global Docker composition setting?

- **Test helper service scope**: The helper service currently handles fund injection and wallet generation. Should it expand to cover other common test operations (e.g., waiting for confirmations, creating tokens), or should those remain in the wallet-lib test helpers?

## Future possibilities

- **Parallel test execution**: With the test helper service managing UTXOs centrally, it becomes feasible to run integration test files in parallel (multiple Jest workers). Each file requests funds independently, and the helper service prevents double-spending.

- **On-demand block production**: Instead of a fixed interval, the dev-miner could expose an API to mine blocks on demand (similar to Bitcoin's `generatetoaddress`). Tests that need confirmations could request exactly the number of blocks they need, making tests faster and more deterministic.

- **Weight-zero mode**: A more aggressive optimization would skip PoW validation entirely in test mode (weight = 0, nonce = 0 always valid). This would eliminate even the trivial nonce iteration, though it further diverges from production behavior.
