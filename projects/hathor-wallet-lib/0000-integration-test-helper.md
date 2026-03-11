- Feature Name: integration-test-helper
- Start Date: 2026-02-24
- Author: Tulio <tulio@hathor.network>)

# Summary

The [Integration Test Helper](https://github.com/tuliomir/hathor-integration-test-helper)
is a dockerized HTTP service that provides race-condition-free wallet
generation and funding for
[hathor-wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)
integration tests. It pre-generates BIP39 wallets with derived addresses
outside the test runtime (avoiding
[prohibitive address derivation costs under Jest](https://github.com/HathorNetwork/hathor-wallet-headless/issues/173))
and splits the genesis block's single UTXO into a pool of independent,
readily-available UTXOs so that parallel test executions never compete
for funds.

# Motivation

The [hathor-wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)
integration test suite currently has two structural blockers that prevent
test parallelization:

## 1. Wallet generation is prohibitively slow under Jest

Deriving addresses from BIP39 seeds under Jest's execution wrapper takes
7 seconds for simple wallets and up to 60 seconds for multisig wallets,
compared to under 500 ms when running outside Jest
([hathor-wallet-headless#173](https://github.com/HathorNetwork/hathor-wallet-headless/issues/173)).
The current workaround is a
[static JSON file with 100 pre-calculated wallets](https://github.com/HathorNetwork/hathor-wallet-lib/blob/master/__tests__/integration/helpers/wallet-precalculation.helper.ts)
that are consumed serially. This works for sequential execution but
breaks under parallelization: concurrent tests drawing from the same
file race to consume wallets, leading to duplicates and test failures.

## 2. All funding derives from a single genesis UTXO

Every integration test that needs funds must ultimately spend from the
same genesis block UTXO. When tests run in parallel, multiple funding
transactions attempt to spend the same UTXO simultaneously, causing
race-condition failures. There is no built-in mechanism in
[hathor-wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)
to coordinate UTXO selection across independent test processes.

## Expected outcome

With this service running alongside the test infrastructure, integration
tests can:

- Instantly obtain unique, ready-to-use wallets with pre-derived
  addresses (no Jest-side derivation).
- Request funding without coordinating with other tests — each request
  gets its own dedicated UTXO, eliminating race conditions.
- Run fully in parallel, reducing total suite execution time
  proportionally to the number of parallel workers.

# Guide-level explanation

## Overview

The Integration Test Helper is a lightweight HTTP service that runs as a
Docker container alongside the existing test infrastructure
([hathor-core](https://github.com/HathorNetwork/hathor-core) fullnode +
[tx-mining-service](https://github.com/HathorNetwork/tx-mining-service)).
It exposes four endpoints:

| Method | Path             | Purpose                                    |
|--------|------------------|--------------------------------------------|
| GET    | `/simpleWallet`  | Get a new simple wallet (seed + addresses) |
| GET    | `/multisigWallet`| Get N-of-M multisig wallets                |
| POST   | `/fund`          | Fund a test wallet address with HTR        |
| GET    | `/status`        | Pool health and readiness                  |

## Wallet generation

When a test needs a wallet, it makes a `GET /simpleWallet` request. The
service returns a 24-word BIP39 seed and 22 pre-derived addresses. Since
address derivation happens inside the service (not under Jest), it
completes in milliseconds. Wallets are pre-generated and cached, so
responses are near-instant.

For multisig tests, `GET /multisigWallet?participants=N&numSignatures=M`
generates N participant wallets that share the same P2SH addresses.

## Funding

When a test needs HTR, it makes a `POST /fund` request with the target
address and optional amount. The service atomically reserves a UTXO from
its internal pool, builds and broadcasts a transaction, and returns the
transaction ID. No two tests ever attempt to receive funds from the same UTXO.

## Typical test workflow

```
1. Start infrastructure:  fullnode + tx-mining-service + test-helper
2. Wait for /status to return { ready: true }
3. For each parallel test:
   a. GET  /simpleWallet         → { words, addresses }
   b. Start wallet with pre-calculated addresses (no derivation)
   c. POST /fund { address, amount }  → { txId }
   d. Wait for wallet to sync the funding tx
   e. Run test assertions
```

Tests are fully isolated: each gets its own wallet and its own funding
UTXO.

## How test developers should think about this

- **Wallet generation is free** — request as many as you need, the cache
  auto-refills.
- **Funding is atomic** — you will never get a "UTXO already spent"
  error from a race condition with another test.
- **The service is stateless from the test's perspective** — no
  registration, no sessions, no cleanup needed.

# Reference-level explanation

## Architecture

The [service](https://github.com/tuliomir/hathor-integration-test-helper)
is built with [Bun](https://bun.sh/) and
[hathor-wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)
and is deployed as a Docker container. It connects to a
[hathor-core](https://github.com/HathorNetwork/hathor-core) fullnode
and a [tx-mining-service](https://github.com/HathorNetwork/tx-mining-service)
instance for transaction PoW.

```
┌─────────────────┐     ┌──────────────┐     ┌───────────────────┐
│  Test Process 1  │────▶│              │────▶│  Hathor Fullnode   │
├─────────────────┤     │  Test Helper │     └───────────────────┘
│  Test Process 2  │────▶│   Service    │           │
├─────────────────┤     │              │     ┌─────▼─────────────┐
│  Test Process N  │────▶│  :3020       │────▶│ TX Mining Service │
└─────────────────┘     └──────────────┘     └───────────────────┘
```

## Wallet generation subsystem

### Seed generation

Uses
[wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)'s
`walletUtils.generateWalletWords()` (BIP39) to produce 24-word seeds.
Addresses are derived at the standard Hathor path `m/44'/280'/0'/0` for
simple wallets. For multisig wallets, each participant gets their own
seed; xpubs are sorted deterministically before deriving shared P2SH
addresses.

Each generated wallet provides 22 pre-derived addresses, enough for
typical test scenarios.

### Caching

Simple wallets are pre-generated at startup (configurable via
`SIMPLE_WALLET_CACHE_SIZE`, default 10) and served from an in-memory
FIFO cache. When the cache is consumed, it refills asynchronously in the
background.

Multisig wallets are generated on-demand since the parameter space
(participants × signatures) is too wide to pre-cache effectively.

## UTXO pool subsystem

### The problem in detail

On a private testnet, the genesis block creates a single large UTXO
(e.g., 1 billion HTR). All test funding must ultimately derive from
this UTXO. If two tests try to spend it simultaneously, one succeeds
and the other fails — a classic double-spend race condition.

### Three-bucket pool design

The service maintains an in-memory pool with three buckets:

1. **testUtxos** (FIFO queue): Fixed-amount UTXOs (~1000 HTR each),
   one per test funding request. This is the primary bucket.
2. **largeUtxo** (single slot): One large UTXO reserved for splitting
   into more test UTXOs or for funding requests that exceed the
   standard amount.
3. **leftoverUtxos** (list): Variable-amount change UTXOs from
   partial funding operations.

### UTXO lifecycle

```
Genesis UTXO (1B HTR)
    │
    ▼ splitUtxo()
┌───────────────────────────────────────┐
│  testUtxos[0]  = 1000 HTR            │
│  testUtxos[1]  = 1000 HTR            │
│  ...                                  │
│  testUtxos[99] = 1000 HTR            │
│  largeUtxo     = remaining (~1B)   │
└───────────────────────────────────────┘
    │
    ▼ fundAddress() consumes testUtxos[0]
┌───────────────────────────────────────┐
│  TX: testUtxos[0] → target + change  │
│  change → leftoverUtxos (if any)     │
└───────────────────────────────────────┘
    │
    ▼ needsRefill() triggers when testUtxos < threshold
┌───────────────────────────────────────┐
│  splitUtxo(largeUtxo) → 100 more     │
│  testUtxos + new largeUtxo           │
└───────────────────────────────────────┘
```

### Race-condition freedom

The pool achieves race-condition freedom through JavaScript's
single-threaded event loop. `reserveUtxo()` performs a synchronous
array `shift()` — the dequeue completes atomically before any other
request handler can execute. No locks, mutexes, or compare-and-swap
operations are needed.

### UTXO classification

UTXOs are categorized by amount relative to the configured split amount:
- **Test-sized**: within ±10% of `UTXO_SPLIT_AMOUNT` → `testUtxos`
- **Large**: the biggest non-test UTXO → `largeUtxo`
- **Leftover**: everything else → `leftoverUtxos`

### Reserve logic

For amounts ≤ `UTXO_SPLIT_AMOUNT`:
1. Dequeue from `testUtxos` (FIFO)
2. Fallback: find first sufficient UTXO in `leftoverUtxos`
3. If both empty: return HTTP 409

For amounts > `UTXO_SPLIT_AMOUNT`:
1. Claim `largeUtxo` if sufficient
2. Otherwise: queue the request with a timeout
   (`FUND_TIMEOUT_MS`, default 30s)
3. Request resolves when a large UTXO becomes available (e.g., after
   a refill), or times out with HTTP 409

### Auto-refill

When `testUtxos.length` drops below `REFILL_THRESHOLD` (default 10)
after a funding operation, the service triggers a background split of
the current `largeUtxo`. The split is delayed by 1.5 seconds to avoid
timestamp collisions with the parent transaction (fullnode requires
`tx.timestamp > parent.timestamp`).

Only one split runs at a time (`splitInProgress` flag).

## Transaction building

All transactions use
[wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)'s
`TransactionTemplateBuilder` API:

1. Create template with explicit raw input (reserved UTXO)
2. Add outputs (target address + optional change to genesis address)
3. Build and sign with configured PIN
4. Broadcast via `SendTransaction.runFromMining()`, which delegates
   PoW to the
   [tx-mining-service](https://github.com/HathorNetwork/tx-mining-service)

The `wallet` instance (not just `storage`) is passed to
`SendTransaction` to ensure `handlePushTx()` calls
`enqueueOnNewTx()`, keeping wallet-lib's internal state in sync.

## Genesis wallet lifecycle

On startup (when `GENESIS_SEED_WORDS` is configured):

1. Connect to fullnode via wallet-lib's `WalletConnection`
2. Start `HathorWallet` instance, poll until ready
3. Scan existing UTXOs and populate the pool
4. If no test UTXOs exist: wait for genesis block reward lock to
   expire (`REWARD_SPEND_MIN_BLOCKS`), then perform initial split
5. Begin serving `/fund` requests

The wallet overrides wallet-lib's hardcoded `TX_WEIGHT_CONSTANTS`
when `TX_MIN_WEIGHT` is set, producing minimal-weight transactions
suitable for private testnets running with `--test-mode-tx-weight`.

## Stale UTXO recovery

External code (e.g., a test spending a UTXO directly on the fullnode)
may invalidate a reserved UTXO. When `runFromMining()` fails with
"already been spent":

1. Trigger `rescanUtxoPool()`: re-query `wallet.getAvailableUtxos()`
2. Repopulate all three buckets from the fresh UTXO set
3. Retry the funding operation once

Concurrent rescans are coalesced into a single query. Incoming fund
requests wait for the rescan to complete before proceeding.

## Docker deployment

The service is packaged as a multi-stage Docker image
(`oven/bun:1` → `oven/bun:1-slim`). A reference `docker-compose.yml`
runs it alongside the fullnode and tx-mining-service with health
checks that gate test execution until the pool is ready.

The service's Docker healthcheck polls `GET /status` and waits for
`{ ready: true }`, which is only returned after the initial UTXO
split completes.

## Dependency on dev-miner mode

This service's reliability on private testnets depends on the
[tx-mining-service](https://github.com/HathorNetwork/tx-mining-service)'s
`--dev-miner` mode, proposed in
[RFC PR #102: Dev Miner for Integration Tests](https://github.com/HathorNetwork/rfcs/pull/102).
The dev-miner mode provides deterministic block mining without a
separate cpuminer process, eliminating timing-dependent test failures
caused by blocks being mined at dozens per second during the initialization of the blockchain.

This condition happens only once per blockchain, but in such short-lived disposable test environments it is a
frequent source of failures that must be fixed.

## Configuration

| Variable               | Default                      | Purpose                               |
|------------------------|------------------------------|---------------------------------------|
| `GENESIS_SEED_WORDS`   | —                            | 24-word BIP39 seed (required for /fund) |
| `HATHOR_NODE_URL`      | `http://localhost:8083/v1a/` | Fullnode REST API                     |
| `TX_MINING_URL`        | `http://localhost:8035/`     | TX mining service URL                 |
| `TX_MIN_WEIGHT`        | —                            | Override wallet-lib weight constants  |
| `PORT`                 | `3020`                       | Server listen port                    |
| `SIMPLE_WALLET_CACHE_SIZE` | `10`                     | Pre-generated wallet cache size       |
| `UTXO_SPLIT_AMOUNT`   | `1000`                       | HTR per test UTXO                     |
| `UTXO_SPLIT_COUNT`    | `100`                        | Max UTXOs per split batch             |
| `REFILL_THRESHOLD`     | `10`                         | Auto-refill trigger threshold         |
| `FUND_TIMEOUT_MS`      | `30000`                      | Large UTXO request queue timeout      |
| `WALLET_PASSWORD`      | `test-password`              | Wallet-lib storage encryption         |
| `WALLET_PIN_CODE`      | `123456`                     | Transaction signing PIN               |

# Drawbacks

- **New infrastructure dependency**: integration tests now require an
  additional Docker service.

- **Single point of failure**: the service is a centralized
  intermediary. If it crashes or becomes unresponsive, all parallel
  tests lose access to new wallets and funding simultaneously.

- **Genesis wallet coupling**: the service must hold the genesis
  wallet's seed, concentrating fund access. This is acceptable for
  testing but reinforces that this tool must never be used in
  production-like environments.

# Rationale and alternatives

## Why this design

- **External service vs. in-process library**: the core problem (Jest
  slowness for address derivation) cannot be solved inside the test
  process. An external service is the only architecture that moves
  derivation outside Jest.

- **UTXO pre-splitting vs. on-demand funding**: pre-splitting
  eliminates race conditions by construction. On-demand funding from
  a shared UTXO would require distributed locking, which is fragile
  and adds latency.

- **Single-threaded JS for atomicity**: rather than introducing
  external coordination (Redis, database locks), the design exploits
  JavaScript's event loop for lock-free atomic UTXO reservation.

## Alternatives considered

1. **Expand the static JSON wallet file**: generate more wallets and
   use file-based locking to coordinate access. Rejected because
   file locking is unreliable across platforms and test frameworks.

2. **In-process UTXO pool**: each test process maintains its own UTXO
   pool. Rejected because it still requires cross-process UTXO
   coordination for the initial split.

3. **Database-backed UTXO ledger**: use SQLite or Redis to track UTXO
   assignments. Rejected as over-engineered for the test environment
   — adds operational complexity without meaningful benefit over the
   single-threaded approach.

## Impact of not doing this

Without this service, test parallelization remains blocked. The test
suite will continue running serially, limiting CI throughput and developer feedback loops.

# Prior art

- **The current
  [WalletPrecalculationHelper](https://github.com/HathorNetwork/hathor-wallet-lib/blob/master/__tests__/integration/helpers/wallet-precalculation.helper.ts)**:
  a static JSON file of 100 pre-generated wallets consumed serially.
  This works for sequential tests but breaks under parallelization.
  The Integration Test Helper replaces and generalizes this approach.

- **[hathor-wallet-headless#173](https://github.com/HathorNetwork/hathor-wallet-headless/issues/173)**:
  the original investigation into Jest performance that identified
  the address derivation bottleneck and motivated external
  pre-calculation.

- **Bitcoin's `regtest` funding pattern**: Bitcoin Core's regtest mode
  uses `generatetoaddress` to create block rewards for test wallets.
  The Integration Test Helper follows a similar philosophy — provide
  a simple, test-oriented funding mechanism — but adapted to Hathor's
  UTXO and mining model.

# Unresolved questions

None

# Future possibilities

- **Custom token support**: extend `/fund` to accept a token UID and
  handle token creation + minting in the UTXO pool.

- **Wallet-lib native integration**: if Jest performance is eventually
  fixed (e.g., via native crypto modules), the wallet generation
  aspect could be folded back into
  [wallet-lib](https://github.com/HathorNetwork/hathor-wallet-lib)
  as a test utility.

- **Multi-node support**: for stress testing scenarios, the service
  could manage UTXO pools across multiple fullnodes.

- **Cached MultiSig wallets**: The Wallet Lib test suite could determine
  a list of desired N and M parameters to preemptively generate and cache,
  further reducing eventual wait times

- **Metrics and observability**: expose Prometheus-like metrics for UTXO
  pool depth, funding latency, and refill frequency to support CI
  performance monitoring.

- **Miner wallet management**: some tests also require interacting with
  the miner rewards, and access its wallet directly. This service could
  provide the same benefits to multiple tests that need to access this wallet.

- **On-Chain Blueprints wallet management**: same idea from the above, but for
  managing the OCB Wallet, with blueprint-specific methods and endpoints.
