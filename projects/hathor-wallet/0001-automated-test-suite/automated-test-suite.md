- Feature Name: automated-test-suite
- Start Date: 2026-05-20
- RFC PR: (to be filed)
- Hathor Issue: (leave this empty)
- Author: Tulio Miranda <tulio@hathor.network>

#### Table of content
[table-of-content]: #table-of-content

- [Summary](#summary)
- [Motivation](#motivation)
- [Guide-level explanation](#guide-level-explanation)
- [Reference-level explanation](#reference-level-explanation)
- [Drawbacks](#drawbacks)
- [Rationale and alternatives](#rationale-and-alternatives)
- [Prior art](#prior-art)
- [Unresolved questions](#unresolved-questions)
- [Future possibilities](#future-possibilities)

# Summary
[summary]: #summary

Introduce a multi-layered automated test suite for the Hathor Desktop Wallet ([HathorNetwork/hathor-wallet](https://github.com/HathorNetwork/hathor-wallet)), an Electron + React + Redux + redux-saga app. The suite has four layers — unit, integration, component, and end-to-end — mirroring the structure of the sibling mobile RFC ([HathorNetwork/rfcs#110](https://github.com/HathorNetwork/rfcs/pull/110)). The E2E layer uses Playwright driving Electron via the Chrome DevTools Protocol — the same path Chrome devtools use — which sidesteps the LavaMoat/SES concerns that drove the mobile RFC to Maestro.

Work lands incrementally: a reference smoke-only PR for Layers 1–3, a second reference smoke-only PR for Layer 4, then one focused PR per feature area for full coverage.

# Motivation
[motivation]: #motivation

The desktop wallet currently has no enforced automated quality gate. Concretely:

1. **Jest exists but is disabled in CI.** `.github/workflows/main.yml` has the `npm test` step commented out, citing a since-resolved react-scripts/React 18 incompatibility. There are 9 ad-hoc tests under `src/__tests__/` that nobody has run in CI for months.
2. **Coverage thresholds are placeholders.** `package.json` declares thresholds of 1–5% — they exist only so the coverage tooling does not error out, not as a real gate.
3. **Cypress was attempted as a PoC, never as a real testing layer.** A single welcome-flow spec exists under `cypress/e2e/00-welcome.cy.js`, exercised by `e2e.yml` against `npm start` (the React app served by webpack-dev-server). It does not test the Electron-packaged app, IPC, main-process code, native dialogs, or hardware-wallet integration.
4. **The release-validation surface is six Markdown checklists.** `QA.md`, `QA_LEDGER.md`, `QA_Nano.md`, `QA_LARGE_VALUES.md`, `QA_WALLET_SERVICE.md`, and `SHORT_QA.md` together describe a multi-hour manual run before each release. QA is the sole correctness signal.
5. **Asymmetry with the mobile wallet.** The sibling mobile wallet's automated-test-suite RFC ([HathorNetwork/rfcs#110](https://github.com/HathorNetwork/rfcs/pull/110)) has the same motivation. Aligning the desktop wallet's testing structure with mobile's lets engineers cross-cut between repos and share patterns.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Testing pyramid

The suite follows a four-layer pyramid:

```
        ┌─────────┐
        │  E2E    │  ← Playwright + Electron: critical user journeys
        │ (few)   │     against the packaged-style app + local docker fullnode
       ┌┴─────────┴┐
       │ Component  │  ← @testing-library/react: screen-level rendering
       │ (moderate) │    with Redux store and mocked navigation
      ┌┴───────────┴┐
      │ Integration  │  ← redux-saga-test-plan: saga + reducer end-to-end
      │ (moderate)   │
     ┌┴─────────────┴┐
      │    Unit        │  ← Jest: pure functions, reducers, utilities
      │ (many, fast)   │
      └───────────────┘
```

### Layer 1 — Unit (Jest)

Pure-function tests for `utils/`, financial math, selectors, and reducer state transitions. Run in milliseconds via the existing CRA test runner.

```bash
npm test
```

Reducer tests pin **three contracts**: initial-state shape with top-level keys alphabetically sorted, exact action-type literal strings, and observable behavior. The three-contract layout is the deliberate safety net for the eventual RTK-slices migration; tests that assert only behavior leave the migration unprotected.

### Layer 2 — Integration (redux-saga-test-plan)

`expectSaga`-style tests that dispatch an action, verify the saga calls the expected services, and assert the resulting reducer state. The wallet has substantial saga surface (`src/sagas/wallet.js`, `tokens.js`, `nanoContract.js`, `reown.js`, `atomicSwap.js`, `networkSettings.js`, `featureToggle.js`, `helpers.js`, `modal.js`); each follow-up feature-area PR adds the sagas it touches.

### Layer 3 — Component (@testing-library/react)

Screen-level rendering tests that mount components wrapped by `renderWithProviders` (Redux store + `MemoryRouter`), simulate user interactions with `@testing-library/user-event`, and assert on the resulting UI and dispatched actions. Helpers live under `__tests__/helpers/`: `renderWithProviders`, `createTestStore` (preloaded-state merge), `mockNavigation` (`react-router-dom` v6), `getInitialState`. A test should rely on these helpers and on the centralized mocks in `jestMockSetup.js` rather than redeclare mocks per file.

### Layer 4 — E2E (Playwright + Electron)

Full user journeys executed against an Electron-launched build of the wallet. Playwright drives the renderer via the Chrome DevTools Protocol — the same mechanism Chrome devtools use — and reaches the main process via `electronApp.evaluate()` for IPC and native-dialog assertions.

CDP is **not** a runtime instrumentation injected into the JS environment; the wallet code under test runs unmodified. This avoids the class of LavaMoat/SES compatibility risk that drove the mobile RFC ([HathorNetwork/rfcs#110](https://github.com/HathorNetwork/rfcs/pull/110)) to pick the gray-box-avoiding Maestro over Detox. The desktop wallet uses `@lavamoat/webpack`, and the renderer running under Playwright still has its normal LavaMoat hardening applied — Playwright observes it from outside via CDP, not from inside the JS sandbox.

## What E2E covers, and what it doesn't

| Covered by E2E | NOT covered by E2E |
|---|---|
| Full navigation flows (wallet creation, send/receive, token mgmt) | Cross-platform native packaging (macOS notarization, Windows signing, Linux AppImage/.deb specifics) |
| Renderer ↔ main IPC paths through `public/electron.js` and `public/preload.js` | Hardware Ledger over real USB |
| Native-dialog interception (file pickers, save dialogs) via main-process eval | Sentry telemetry behavior |
| Multi-modal flows (lock screen, PIN modals, transaction overview modals) | Wallet Service mode (deferred to future possibilities) |
| Live blockchain interaction (against `qa/large-values-network/` docker network) | — |

QA remains essential for the right-hand column. E2E takes over the repetitive left-hand column, freeing QA to focus on exploratory testing, hardware paths, and packaging.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## PR sequencing

This RFC ships in two reference PRs followed by an open-ended sequence of feature-area PRs.

### PR 1 — Reference smoke for Layers 1, 2, 3 + infrastructure

Scope:
- Test layout under top-level `__tests__/` (existing `src/__tests__/` files migrate).
- Helpers under `__tests__/helpers/`.
- `jestMockSetup.js` with centralized mocks for `@hathor/wallet-lib`, `@hathor/hathor-rpc-handler`, `@reown/walletkit`, `@walletconnect/*`, `@sentry/electron`, `@ledgerhq/hw-transport-node-hid`, `unleash-proxy-client`, `ttag`, and the `electron` IPC bridge.
- The reference smoke set defined below — one canonical example per testing technique.
- Re-enable the currently commented-out `npm test` step in `.github/workflows/main.yml` (see [Reactivating Jest in CI](#reactivating-jest-in-ci)).
- Initial agent / contributor docs (see [AI agent / contributor documentation](#ai-agent--contributor-documentation)): `AGENTS.md`, `CLAUDE.md`, `.claude/skills/writing-tests/SKILL.md`, `docs/testing-guide.md`.

Out of scope for PR 1: any Layer-4 work, any docker network changes, Playwright, the `cypress/` directory.

### PR 1 smoke set

The reference smoke is not "one test per layer." It is **one test per testing technique** future feature-area PRs will need. Each entry below establishes a copy-paste-and-extend pattern; the file paths are illustrative (the implementer picks the specific source-side function or screen that best fits each pattern).

| Layer | Test file (indicative) | Pattern demonstrated |
|---|---|---|
| 1 (utility) | `__tests__/utils/tokens.test.ts` | Pure-function test with no mocks, no state, no async — baseline Jest pattern |
| 1 (selector) | `__tests__/utils/walletSelectors.test.ts` | Function consuming Redux state — how to construct a test state object and pass it in |
| 1 (reducer) | `__tests__/reducers/reducer.wallet.test.ts` | Three-contract reducer test (initial-state shape + action-type literals + behavior) |
| 2 (saga) | `__tests__/sagas/wallet.test.ts` | `expectSaga` with `provide()` injecting a mocked wallet-lib response — saga + reducer integration with mock injection at the saga boundary |
| 2 (saga state reset) | `__tests__/sagas/atomicSwap.test.ts` | Saga test demonstrating the `*ForTesting` module-state-reset pattern — module-state hygiene |
| 3 (happy path) | `__tests__/screens/Welcome.test.tsx` | Screen rendering with preloaded Redux state — `renderWithProviders` + `createTestStore` |
| 3 (error path) | `__tests__/screens/NewSoftwareWallet.test.tsx` | Asserting a non-happy-path UI state — form validation failure or error modal display |
| 3 (navigation) | `__tests__/screens/Settings.test.tsx` | User action triggering navigation — `mockNavigation` usage and route-change assertion |

After PR 1 lands, any feature-area PR can write its tests by finding the matching entry above and following its file shape. Reviewer cognitive load on later PRs becomes `O(coverage)`, not `O(coverage + infrastructure + pattern-discovery)`.

### PR 2 — Reference smoke for Layer 4

Scope:
- Playwright + `_electron.launch()` configuration (`playwright.config.ts`, `e2e/fixtures/electronApp.ts`).
- Two E2E flows that together cover both reference shapes:
  - **Welcome flow** — Playwright reimplementation of the assertions in the previous `cypress/e2e/00-welcome.cy.js` PoC. No backend interaction; useful as the minimal Electron-app smoke.
  - **Large Values flow** — exercises the live blockchain test path against `qa/large-values-network/`. Replaces the current human-driven `qa/large-values-network/checks.py` with JS assertions in the test file itself, using targeted queries rather than the python script's whole-chain-state expectations.
- CI wiring for the local docker network in `.github/workflows/e2e.yml` (replacing the Cypress job).
- Removal of the `cypress/` directory, `cypress.config.ts`, and Cypress devDependencies; explicit note that Cypress was a PoC, not an implementation.
- Extensions to the agent docs landed in PR 1 — Playwright-specific patterns, `electronApp.evaluate()` conventions, Linux CI `xvfb-run` notes (see [AI agent / contributor documentation](#ai-agent--contributor-documentation)).

Decisions deferred to PR 2:

- **Electron `userData` isolation across Playwright test runs.** `_electron.launch()` accumulates `userData` state across launches unless each test points at an isolated temp dir. The exact fixture shape (per-test temp directory vs. per-suite + manual reset) is settled when the Playwright config is written.
- **Final `qa/large-values-network/` layout.** The directory's docker-compose is the right shape, but its `checks.py` and `privnet.yml` are part of the legacy human-driven QA flow. Whether PR 2 carves out a sibling `qa/e2e-network/` or extends the existing directory in place is decided alongside the CI wiring.

Out of scope for PR 2: any per-feature E2E coverage beyond the two reference flows.

### PRs 3..N — Feature-area suites

Each follow-up PR owns one slice of the application and ships its full Layer-1-through-4 coverage where applicable. The slice anchors are the existing manual QA documents:

| Feature area | Anchor doc | Likely layers |
|---|---|---|
| New wallet creation / backup words | QA.md "Initialization" | 1+2+3+4 |
| Lock / Unlock | QA.md | 1+2+3 |
| Send Tokens (incl. fee model) | QA.md "Send Tokens" | 1+2+3+4 |
| Custom Tokens (create / register / admin) | QA.md "Create new token" | 1+2+3+4 |
| Token Import (banner + modal) | QA.md "Token import" | 1+2+3 |
| NFT creation | QA.md | 1+2+3 |
| Single Address Mode | (new in master) | 1+2+3 |
| Atomic Swap | QA.md | 1+2 |
| Reown / WalletConnect | (sagas/reown.js) | 1+2+3 |
| Nano Contracts | QA_Nano.md | 1+2+3+4 |
| Ledger / Hardware Wallet | QA_LEDGER.md | 1+2+3 (JS-mock) |
| Address management | QA.md "Addresses" | 1+2+3 |

PR 1 establishes the bar; every follow-up PR is a copy-paste-and-extend of the same patterns.

## Test layout

```
__tests__/
├── helpers/
│   ├── renderWithProviders.tsx     # Redux store + MemoryRouter wrapper
│   ├── createTestStore.ts          # preloaded-state merge
│   ├── mockNavigation.ts           # react-router-dom v6 mocks
│   ├── getInitialState.ts          # root reducer initial state
│   └── ledgerTransportMock.ts      # scripted APDU responses; top-of-file
│                                   # comment points at hathor-ledger-app
├── utils/
│   ├── tokens.test.ts              # ← L1 smoke: pure function
│   └── walletSelectors.test.ts     # ← L1 smoke: selector / state-consuming
├── reducers/
│   └── reducer.wallet.test.ts      # ← L1 smoke: reducer three contracts
├── sagas/
│   ├── wallet.test.ts              # ← L2 smoke: expectSaga + provide()
│   └── atomicSwap.test.ts          # ← L2 smoke: *ForTesting state reset
└── screens/
    ├── Welcome.test.tsx            # ← L3 smoke: happy path
    ├── NewSoftwareWallet.test.tsx  # ← L3 smoke: error path
    └── Settings.test.tsx           # ← L3 smoke: navigation

jestMockSetup.js                    # setupFiles entry; centralized mocks

e2e/                                # ← PR 2 onward (Playwright)
├── playwright.config.ts
├── fixtures/
│   └── electronApp.ts              # _electron.launch() helper
└── flows/
    ├── welcome.spec.ts             # ← PR 2 smoke
    └── largeValues.spec.ts         # ← PR 2 smoke (uses docker network)
```

The 9 existing `src/__tests__/` files are moved into the appropriate sub-folder above as part of PR 1. The PR description enumerates the old→new path mapping so reviewers can scan the move.

## Test dependencies (devDependencies only)

| Layer | Package | Notes |
|---|---|---|
| 1, 2, 3 | `@testing-library/react@^14` | Already in devDependencies |
| 1, 2, 3 | `@testing-library/user-event@^14` | Already in devDependencies |
| 1, 2, 3 | `@testing-library/jest-dom@^6` | New — DOM matchers |
| 2 | `redux-saga-test-plan@^4` | New — saga integration |
| 1, 2, 3 | `jest-circus@^29`, `@types/jest@^29` | Runner ergonomics |
| 4 | `@playwright/test@^1` | New — Electron-driving E2E |

Removed in PR 2: `cypress`, `@testing-library/cypress`, `eslint-plugin-cypress`. No production dependencies are added.

## Centralized mocks (`jestMockSetup.js`)

The mocks every test would otherwise re-declare. A new test must rely on these defaults; redeclaring `jest.mock` for one of these in a test file is a review nit captured in the writing-tests skill.

- `@hathor/wallet-lib` sub-path imports — minimal scriptable fakes for the API surface the wallet consumes.
- `@hathor/hathor-rpc-handler` — fake handler that records calls.
- `@reown/walletkit`, `@walletconnect/core`, `@walletconnect/utils` — no-op stubs (these libraries spin up WebSocket connections).
- `@sentry/electron` — no-op.
- `@ledgerhq/hw-transport-node-hid` — scripted transport that returns canned APDU responses. The mock file's top comment points at [HathorNetwork/hathor-ledger-app](https://github.com/HathorNetwork/hathor-ledger-app) as the canonical source for the response shapes; when the real device behavior diverges, this comment is the entry point.
- `unleash-proxy-client` — returns fixed feature-flag values. (If this repo later migrates to `@hathor/unleash-client` to align with mobile, the centralized mock follows the migration.)
- `ttag` translations — identity function so assertions can match English literals.
- `electron` module — fake `ipcRenderer` since Jest runs in Node without an Electron runtime; provides scripted IPC responses.

## Helpers

- **`renderWithProviders(ui, options)`** — wraps a component in `Provider` (with a preloaded test store) and `MemoryRouter` (with optional `initialEntries`). Returns the standard RTL render result plus the store reference.
- **`createTestStore(preloadedState)`** — merges a partial preloaded state with `getInitialState()`, configures the same middlewares as production (redux-thunk + redux-saga), returns the store.
- **`mockNavigation()`** — returns a `useNavigate` / `useLocation` / `useParams` mock set, ready for `jest.mock('react-router-dom', …)`.
- **`getInitialState()`** — calls the root reducer with `undefined` state and a `@@INIT`-style action, returns the result. The reducer-test three-contract layer asserts on the shape of this return value.
- **`ledgerTransportMock`** — scripted transport whose default responses match the [hathor-ledger-app](https://github.com/HathorNetwork/hathor-ledger-app) happy paths; per-test overrides via a fluent `.respondTo(apdu, response)` API.

## Production code changes expected

Expected production-code changes for the test suite are small and bounded. The Chromium renderer driven via CDP imposes no testability-driven changes on the renderer itself; the expected modifications are limited to:

| Class | Likelihood | Example |
|---|---|---|
| Test-only `*ForTesting` named exports to reset module-level saga state | 1–3 across PRs | `clearAtomicSwapStateForTesting` in `sagas/atomicSwap.js` |
| `data-testid` props on interactive elements that lack stable selectors | Added per-feature-PR as needed | The Cypress PoC used a mix of placeholders and CSS ids; Playwright tests prefer explicit `data-testid` |
| Disable Sentry and feature-flag polling under `HATHOR_TEST=1` | One place each in PR 2 | `public/electron.js`, `src/sagas/featureToggle.js` |
| Renderer-specific workarounds | **None expected** | — |

Each `*ForTesting` export is a one-line named export marked test-only by convention.

## Ledger strategy

PR 1 adds a JS-level transport mock at the boundary of `@ledgerhq/hw-transport-node-hid`. The mock returns scripted APDU responses; the wallet's `public/ledger.js` (main-process) and the Ledger-aware sagas/components are exercised against it without modification. Real-device coverage stays with `QA_LEDGER.md`.

[HathorNetwork/hathor-ledger-app](https://github.com/HathorNetwork/hathor-ledger-app) is referenced in two places:

1. In this RFC, as the canonical source for the APDU contract that the mock simulates.
2. At the top of `__tests__/helpers/ledgerTransportMock.ts`, as a `// See: …` comment that tells a future contributor where to look when the mock starts disagreeing with the real device.

Speculos-based emulation (run the real `hathor-app.elf` binary inside a docker container, drive it from Node via `@ledgerhq/hw-transport-http`) is documented as a future possibility. It would lift the JS-mock's fidelity ceiling and align this wallet's Ledger tests with how hathor-ledger-app itself is tested, but it's a non-trivial CI investment and out of scope here.

## Local docker network strategy

`qa/large-values-network/docker-compose.yml` exists in the repo today (fullnode + tx-mining-service + cpuminer), but has never been wired into CI. PR 2 is its first CI exercise. Before becoming usable for the broader test suite it needs:

- The `cpuminer` service replaced with `tx-mining-service`'s built-in `dev-miner` feature, mirroring the configuration the wallet-lib integration tests already use.
- Integration with the [integration-test-helper](https://github.com/HathorNetwork/hathor-integration-test-helper) — the dockerized HTTP service specified in [RFC 103](../../hathor-wallet-lib/0000-integration-test-helper.md) (under development at the time of writing). It provides race-condition-free wallet generation and pre-split UTXO pools so parallel test executions never compete for funds. Desktop E2E consumes it directly rather than re-inventing the primitive.

Assertions against the live network are made in JS in the test file itself, replacing the legacy `qa/large-values-network/checks.py` script. The python script was designed for a human-guided run where the network's exact transaction count was knowable and assertable; the JS replacement uses **targeted queries** ("is this specific transaction confirmed?", "what is this specific address's balance?") rather than whole-chain-state assertions, so the tests remain valid when the network is reused across runs or shared with parallel work.

## AI agent / contributor documentation

PR 1 ships the documentation backbone alongside the smoke tests. PR 2 extends it with the E2E-specific material (Playwright patterns, `electronApp.evaluate()` conventions, Linux CI `xvfb-run` notes). Feature-area PRs improve it whenever they introduce a pattern the existing docs don't cover. The docs are deliberately a first-class deliverable of the reference PRs — not a follow-up — because the whole point of the reference PRs is to reduce future cognitive load.

The four files:

- **`AGENTS.md`** at repo root — cross-tool entry point. Brief authoritative rules pointing to deeper docs.
- **`CLAUDE.md`** at repo root — Claude-Code-specific pointer; same rule set as `AGENTS.md` plus a note on how to load the writing-tests skill.
- **`.claude/skills/writing-tests/SKILL.md`** — auto-loads when an agent edits a test file. Long-form, on-demand depth.
- **`docs/testing-guide.md`** — human-readable reference, also linked from `CONTRIBUTING.md`. Long-form, always available.

### `CLAUDE.md` content (brief, authoritative)

Kept under ~40 lines:

```markdown
## Testing rules

- **Reference vs feature-area distinction.** PRs labelled "reference smoke" land one representative test per technique plus infrastructure. PRs labelled "feature-area" cover one slice of the wallet across all applicable layers. Do not mix the two.
- **Use centralized mocks.** All shared mocks live in `jestMockSetup.js`. Do not redeclare a mock in a test file when a centralized one exists; if you need a different shape, override it locally — do not add a competing mock.
- **`*ForTesting` named exports** — do not remove or rename. Each one resets module-level state that integration tests depend on for isolation. Their `ForTesting` suffix marks them as test-only.
- **Ledger mock is a contract, not a reimplementation.** The transport mock at `__tests__/helpers/ledgerTransportMock.ts` mimics the response shapes documented in https://github.com/HathorNetwork/hathor-ledger-app. JS-mock tests certify wallet code's behavior **given a Ledger response shape**, not Ledger correctness itself; release validation for Ledger flows stays with `QA_LEDGER.md`.
- **Three-contract reducers.** New reducer tests pin (1) initial-state shape, (2) action-type literal strings, (3) behavior. The shape and action-type contracts are the safety net for any future slice/RTK migration.

For deeper guidance: load the writing-tests skill at `.claude/skills/writing-tests/SKILL.md`, or read the long-form reference at `docs/testing-guide.md`.
```

### Skill / knowledge file content

Sections in `.claude/skills/writing-tests/SKILL.md`:

1. Pyramid layout and where each layer's files live.
2. The reference-smoke vs feature-area-PR distinction with worked examples drawn from PRs 1 and 2.
3. Test patterns — one section per entry in the PR 1 smoke set: pure function, selector, reducer three-contracts, saga with `provide()`, saga with `*ForTesting`, screen happy path, screen error path, screen navigation.
4. Ledger JS-mock conventions and the explicit boundary at hathor-ledger-app.
5. Saga test patterns with `redux-saga-test-plan`'s `provide()`, including the module-level-state hygiene rule and the `*ForTesting` pattern.
6. Live-network assertion conventions: targeted queries, not whole-chain-state expectations; teardown discipline.
7. What NOT to test in E2E: packaged-build behavior, real-network conditions, OS-level dialogs, visual regressions, real Ledger hardware.
8. Diagnostic workflow when a Playwright test "fails to find" an element — renderer vs main process, IPC timing, `data-testid` propagation through React portals.

### Why this matters

AI agents working on this codebase will frequently add new screens, modify existing ones, or refactor shared components. Without this guidance, a well-intentioned agent will: skip writing tests because nothing told it that's required; redeclare a mock that the centralized setup already provides, breaking shared-state assumptions; remove a `*ForTesting` export because it "looks like a smell"; write an E2E test that asserts on whole-chain-state and breaks the first time the network is shared; or attempt to test Ledger correctness against the JS-mock when the mock is only a contract simulator.

Each of those individually creates a new review nit cycle. The CLAUDE.md rules plus the skill file with the diagnostic workflow prevent the recurring nit by making the rules discoverable at the right level of detail.

## CI integration

### Reactivating Jest in CI

The `npm test` step in `.github/workflows/main.yml` is currently commented out citing an older react-scripts / React 18 incompatibility. `react-scripts@5.0.1` is now in `dependencies` and the 9 existing tests under `src/__tests__/` are expected to be runnable on a fresh clone.

The first task in PR 1 verifies this. If `npm test` runs the 9 existing tests green, the comment in `main.yml` is stale and the step is simply re-enabled. If not, the fix lands in PR 1 — most likely a small migration of the test setup (e.g., the `--openssl-legacy-provider` flag pattern already used by `start-js` / `build-js`, or a Jest `transformIgnorePatterns` adjustment for React 18 ESM-only dependencies).

### `main.yml` (extended in PR 1)

Re-enables the currently commented-out test step:

```yaml
- name: Run unit + integration + component tests
  run: npm test -- --ci --coverage --maxWorkers=2
- name: Upload coverage
  uses: codecov/codecov-action@<pinned-sha>
```

Zero new infrastructure cost — runs on the existing `ubuntu-latest` runner.

### `e2e.yml` (rewritten in PR 2)

```yaml
- name: Boot local Hathor network
  run: docker compose -f qa/large-values-network/docker-compose.yml up -d --wait # Sample path, will be a dedicated one later
- name: Build wallet for Electron test
  run: npm run build
- name: Run Playwright Electron E2E
  run: xvfb-run -a npx playwright test
- name: Tear down network
  if: always()
  run: docker compose -f qa/large-values-network/docker-compose.yml down -v
```

Replaces the `cypress-io/github-action` step. Linux-only initially (single `ubuntu-latest` runner); cross-platform E2E is a future possibility.

### Coverage thresholds

PR 1 leaves the current 1–5% threshold placeholders in `package.json`. A ratchet to meaningful values is explicitly a future-possibilities item, gated on the first few feature-area PRs landing. **The exact percentages are decided in the first feature-area PR that enforces them** — 25% overall and 90% on financial utilities is a sensible starting frame, but the precise numbers wait until there is enough coverage data to know what is realistic without creating friction.

# Drawbacks
[drawbacks]: #drawbacks

1. **JS-level Ledger mocks drift from real device behavior.** This is the explicit decision; Speculos was evaluated and deferred. Mitigation: `QA_LEDGER.md` remains the gating doc for releases touching Ledger code, and the writing-tests skill states explicitly that JS-mock tests certify wallet code's behavior **given a Ledger response shape**, not Ledger correctness itself. The two-place pointer to hathor-ledger-app (RFC and mock file) provides the contract-drift escape hatch.
2. **The local docker network is not yet CI-tested.** `qa/large-values-network/` has never run on a hosted runner, uses `cpuminer` (slow; needs replacement with `dev-miner`), and depends on the integration-test-helper from [RFC 103](../../hathor-wallet-lib/0000-integration-test-helper.md) being available in CI. PR 2's first CI exercise of the network is likely to expose flakiness. Mitigation: PR 2's Welcome flow does not need the network, so the framework lands even if the Large Values flow needs follow-up tuning.
3. **Smoke-tests-first defers real coverage.** Until feature-area PRs land, the safety net is mostly aspirational. A bug in `send tokens` is no more likely to be caught in PR 1 than today. Mitigation: this is the explicit trade against review-load — and the planned feature-area PRs are scoped from the most-incident-prone paths in the manual QA docs.
4. **The 9 pre-existing tests have no documented contracts.** They were written ad-hoc and have been silent in CI for months. Migrating them in PR 1 should be a near-mechanical path-move, but reviewers will see them in the diff under their new paths. Mitigation: PR 1's description calls out which existing files moved where.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Playwright driving Electron (selected)

Playwright officially supports Electron via `_electron.launch()`. The same Playwright test can drive the renderer (any Chromium-based assertions and interactions) **and** reach into the main process via `electronApp.evaluate()` for IPC, native dialogs, menu bar, and tray-icon assertions.

**What makes it the right choice for this project:**

- **LavaMoat/SES-safe by construction.** Playwright drives the renderer via CDP — the same protocol Chrome devtools use. It never injects instrumentation into the JS runtime, so the wallet's `@lavamoat/webpack` hardening continues to apply unmodified.
- **Main-process access.** Unlike Cypress (renderer-only), Playwright reaches into main-process code, IPC handlers, and the modules under `public/`. This is the surface Electron-specific bugs live on, and the surface manual QA exercises today.
- **Single-runner CI.** Linux `ubuntu-latest` is sufficient (Electron Linux build + `xvfb-run`). Cross-platform E2E (macOS, Windows) is deferred as a future possibility.
- **Mature ecosystem.** Microsoft-maintained, broad community adoption for Electron testing in 2025–2026 (VS Code, GitHub Desktop, Postman use it).

**Weaknesses to be aware of:**

- Slower per-test setup than a unit-test runner; mitigated by keeping E2E count small.
- `electronApp.evaluate()` runs serialized code in main; complex assertions read awkwardly.

## Cypress (PoC, not selected)

Cypress was attempted earlier as a PoC. A single welcome-flow spec at `cypress/e2e/00-welcome.cy.js` runs against `npm start` (the React app served by webpack-dev-server). It was never adopted as the actual testing layer.

The defining limitation for this project is that Cypress drives the renderer only, against the React app served by webpack-dev-server — not the Electron app. Anything in `public/electron.js`, `public/preload.js`, `public/ledger.js`, IPC handlers, native dialogs, main-process Ledger transport, or any other main-process surface is invisible to it. Since most QA-doc-described breakage classes live in those surfaces, Cypress would have been useful only as a renderer-level smoke layer — for which Playwright already suffices.

The existing Cypress directory and devDependencies are removed in PR 2.

## WebdriverIO (not evaluated hands-on)

WebdriverIO + Spectron historically held the Electron-E2E niche but Spectron is deprecated upstream. WebdriverIO's current Electron support is viable but adds a second large client-server runtime to the dev environment without offering anything Playwright does not.

## JS-mock vs Speculos for Ledger

Speculos (the official Ledger emulator) runs the real `hathor-app.elf` binary in QEMU and exposes the device over a TCP socket (port 9999) plus an HTTP automation API (port 5000). hathor-ledger-app's own test suite uses it: pytest tests drive the emulator via `app_client/transport.py`, and CI uses `ghcr.io/ledgerhq/speculos:latest` plus `ghcr.io/ledgerhq/ledger-app-builder` to build the `.elf`.

For the desktop wallet, swapping `@ledgerhq/hw-transport-node-hid` for `@ledgerhq/hw-transport-http` pointed at a Speculos container would exercise the wallet's APDU code end-to-end against the actual ledger binary. This is strictly higher fidelity than the JS-mock and aligns the desktop wallet's Ledger surface with hathor-ledger-app's own test surface.

**Why JS-mock was selected anyway:**

- Speculos adds a docker dependency and a pinned `.elf` artifact to fetch, with non-trivial CI plumbing.
- Layer 1–2 Ledger tests overwhelmingly want to assert *"saga dispatched ACTION_X when ledger returned 0x9000"* — a coarse contract that JS-mocks satisfy without paying full emulator cost.
- The contract-drift risk is real but bounded: the RFC + mock-file pointer to hathor-ledger-app make divergences detectable and traceable.
- Real-device coverage via `QA_LEDGER.md` remains in place as the release gate.

Speculos remains a documented future possibility (see below).

## Impact of not doing this

Without automated tests, every feature addition, dependency update, and refactor carries the risk of undetected regressions. The release-validation surface stays manual, scaling linearly with feature count.

# Prior art
[prior-art]: #prior-art

- **[HathorNetwork/rfcs#110](https://github.com/HathorNetwork/rfcs/pull/110) — Mobile wallet automated test suite.** Direct sibling RFC. The four-layer pyramid, three-contract reducer pattern, AI-agent doc layout, and the reference-then-feature-area PR cadence are aligned with that RFC. The framework choice differs (Playwright + CDP here, Maestro there) for LavaMoat/SES-related reasons explained in the [Layer 4 description](#layer-4--e2e-playwright--electron), and the desktop suite adds an explicit Ledger transport mock that the mobile wallet has no equivalent surface for.
- **[hathor-wallet-lib integration tests](https://github.com/HathorNetwork/hathor-wallet-lib/tree/master/__tests__/integration)** provide deterministic-wallet helpers and the `qa/large-values-network/`-style docker-compose pattern. The desktop suite consumes these patterns directly rather than re-inventing them.
- **[Integration Test Helper](https://github.com/HathorNetwork/hathor-integration-test-helper)** — the dockerized HTTP service designed in [RFC 103](../../hathor-wallet-lib/0000-integration-test-helper.md). Pre-generates BIP39 wallets and splits the genesis UTXO into a pool so parallel test executions never compete for funds. Desktop E2E uses it directly.
- **[hathor-ledger-app pytest + Speculos suite](https://github.com/HathorNetwork/hathor-ledger-app/tree/master/tests)** establishes the contract the desktop's JS-mock simulates. If the JS-mock starts disagreeing with real-device behavior, that repository's tests are the authoritative source.
- **The Electron + Playwright community standard for 2025–2026** has converged on `_electron.launch()` for desktop-app E2E (VS Code, GitHub Desktop, Postman, Slack desktop), replacing the older Spectron + Mocha stack.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

No open questions block RFC approval. Three implementation-level choices are deliberately postponed to the PR that needs them and are documented inline:

- Electron `userData` isolation between Playwright runs — decided in PR 2 (see [PR 2 — Reference smoke for Layer 4](#pr-2--reference-smoke-for-layer-4)).
- Final `qa/large-values-network/` directory layout — decided in PR 2 (see [PR 2 — Reference smoke for Layer 4](#pr-2--reference-smoke-for-layer-4)).
- Coverage-threshold percentages — decided in the first feature-area PR that enforces them (see [Coverage thresholds](#coverage-thresholds)).

# Future possibilities
[future-possibilities]: #future-possibilities

1. **Speculos-based Ledger emulation.** Swap the JS transport mock for `@ledgerhq/hw-transport-http` pointed at a `ghcr.io/ledgerhq/speculos:latest` container running the real `hathor-app.elf`. Aligns the desktop wallet's Ledger surface with hathor-ledger-app's own test surface.
2. **Wallet Service mode coverage.** The wallet has a wallet-service mode toggle (`QA_WALLET_SERVICE.md`). Feature-area PRs that exercise it will need a wallet-service client mock or a wallet-service docker container alongside the fullnode.
3. **Visual regression testing.** Tools like Percy or Applitools screenshot-compare screens across builds, catching layout regressions functional tests miss.
4. **Packaged-build E2E.** Run the smoke flow against the actual `electron-builder`-packaged artifact, exercising the production bundle behavior that `npm run build` + `electron .` does not fully replicate.
5. **Cross-platform E2E.** Run the E2E job on macOS and Windows runners in addition to Linux, catching platform-specific Electron behavior. Adds runner cost.
6. **Coverage threshold ratchet.** Move `package.json` coverage thresholds from the current 1–5% placeholders to meaningful values once enough feature-area PRs have landed.
7. **TDD workflow.** Once the test infrastructure is mature, new features can be developed test-first — writing the component test and the E2E flow before the implementation.
