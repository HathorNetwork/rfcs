- Feature Name: automated-test-suite
- Start Date: 2026-04-17
- RFC PR: https://github.com/HathorNetwork/rfcs/pull/110
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

Introduce a multi-layered automated test suite for the Hathor Mobile Wallet, covering unit tests, integration tests, component tests, and end-to-end tests on iOS/Android simulators. The E2E layer uses Maestro, a black-box testing framework that operates entirely outside the app runtime, preserving compatibility with the app's LavaMoat/SES security hardening.

# Motivation
[motivation]: #motivation

The Hathor Mobile Wallet has grown to 42 screens, 54 components, and 15 saga files handling wallet operations, token management, WalletConnect, nano contracts, and more. Currently, the only automated quality gate is ESLint — tests are not run in CI and manual QA is the sole validation for correctness. This creates several problems:

1. **Regressions go undetected until QA.** Changes to shared utilities (e.g., financial math in `utils.js`) or reducer logic can silently break features across the app.
2. **QA bottleneck.** Every release requires a full manual pass through critical user flows (wallet creation, transactions, backup/restore), which scales poorly as features are added.
3. **Developer confidence.** Without tests, developers cannot refactor or update dependencies with confidence that existing behavior is preserved.

The goal is to establish a testing pyramid that progressively gives developers confidence from fast unit tests (seconds) to full E2E flows on simulators (minutes), with CI integration so that regressions are caught before code review.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Testing pyramid

The test suite follows a four-layer pyramid:

```
         ┌─────────┐
         │  E2E    │  ← Maestro: critical user journeys on simulator
         │ (few)   │     Catches: navigation breaks, integration failures
        ┌┴─────────┴┐
        │ Component  │  ← RNTL: screen-level rendering with user interactions
        │ (moderate) │    Catches: UI regressions, interaction bugs
       ┌┴───────────┴┐
       │ Integration  │  ← redux-saga-test-plan: saga + reducer end-to-end
       │ (moderate)   │    Catches: state management bugs, async flow errors
      ┌┴─────────────┴┐
      │    Unit        │  ← Jest: pure functions, reducers, utilities
      │ (many, fast)   │    Catches: logic errors, edge cases, math bugs
      └───────────────┘
```

### Layer 1: Unit tests (Jest)

Pure function tests for financial math utilities, string helpers, and reducer state transitions. These run in milliseconds and should be written for every new utility or reducer action.

```bash
npm test
```

### Layer 2: Integration tests (redux-saga-test-plan)

Saga-reducer integration tests using `expectSaga`. These verify that dispatching an action runs the correct saga, calls the expected services, and produces the expected state transitions.

### Layer 3: Component tests (React Native Testing Library)

Screen-level rendering tests that mount components with a Redux store and mocked navigation, then simulate user interactions (`fireEvent.press`, `fireEvent.changeText`) and assert on the resulting UI and dispatched actions.

The PoC includes component tests for the BackupWords screen that validate **both** the happy path (selecting correct words advances to the next step) **and** the error path (selecting a wrong word shows the failure modal with "Wrong word." text and navigates back on dismiss).

### Layer 4: E2E tests (Maestro)

Full user flows executed on a running iOS or Android simulator. Maestro interacts with the app through the OS accessibility layer — the same API used by screen readers — and never injects code into the app's JavaScript runtime.

```bash
# Requires: simulator running + Metro bundler serving the app
maestro test .maestro/flows/walletCreation.yaml
```

## What the E2E covers and what it doesn't

| Covered by E2E | NOT covered by E2E |
|---|---|
| Full navigation flows (wallet creation, 5 screens) | Cross-device rendering (Galaxy Fold, old iPhones) |
| Native module integration (Keychain, AsyncStorage) | Release build crashes (Hermes bytecode, ProGuard) |
| Component state (switch toggles, PIN input) | Real network conditions (3G, offline, timeout) |
| React Navigation stack transitions | OS permissions (camera, biometry, notifications) |
| Word backup validation (5-step flow) | Visual regressions (layout shifts, truncation) |

Human QA remains essential for the right column. E2E automation takes over the repetitive left column, freeing QA to focus on exploratory testing and edge cases.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Production code changes

The app requires several changes to work with E2E automation tools on React Native 0.77 + New Architecture (Fabric). These changes stem from how Fabric handles the accessibility tree and touch event delivery via XCUITest, and they are **framework-agnostic** — the same workarounds are required by any XCUITest-based tool.

### 1. Custom ToggleSwitch component

**Problem:** React Native's native `<Switch>` (UISwitch on iOS) does not fire `onValueChange` when tapped programmatically by XCUITest-based tools under Fabric. The native control receives the tap but Fabric's event pipeline does not dispatch the value change to the JS layer. Related issues:
- [wix/Detox#4803](https://github.com/wix/Detox/issues/4803) — isReady timeout with RN 0.76+
- [facebook/react-native#43648](https://github.com/facebook/react-native/issues/43648) — Missing accessibilityLabel on iOS for Switch in Fabric

**Solution:** A custom `ToggleSwitch` component built with `Pressable` that visually mimics the native switch. Props are API-compatible with the native Switch (`value`, `onValueChange`, `trackColor`, `testID`).

### 2. NewHathorButton: TouchableOpacity to Pressable

**Problem:** `TouchableOpacity` (from `react-native`) does not respond to XCUITest taps in Fabric mode. The component is found in the accessibility tree but `onPress` never fires. Related issues:
- [facebook/react-native#44610](https://github.com/facebook/react-native/issues/44610) — Pressable gets stuck under new architecture
- [facebook/react-native#34999](https://github.com/facebook/react-native/issues/34999) — onPress not firing occasionally
- [software-mansion/react-native-screens#2219](https://github.com/software-mansion/react-native-screens/issues/2219) — Pressable elements not working with new architecture on native-stack

**Solution:** Replace with `Pressable`. Additionally, the native `disabled` prop on Pressable prevents XCUITest from registering taps even after the prop changes from `true` to `false`. The disabled behavior is moved to a JS-level guard in `onPress`.

### 3. Flex container layout fix

**Problem:** Interactive elements (buttons) placed inside a `View` with `flex: 1, justifyContent: 'flex-end'` are unreachable by XCUITest. The empty flex space above the button intercepts the tap. This is documented in the [Maestro React Native guide](https://docs.maestro.dev/get-started/supported-platform/react-native) as the "swallowed touch events" issue for deeply nested iOS components.

**Solution:** Move buttons outside the flex container as siblings. Use `<View style={buttonView} />` as an empty spacer, with buttons rendered after it. The visual layout is identical.

### 4. testID props

`testID` props added to: terms agreement switch, Start button, seed words data element, backup step number display, and NumPad buttons. These enable E2E tools to target elements by stable identifier instead of fragile text matching.

## Wallet creation E2E flow

The PoC implements the complete wallet creation journey:

1. **WelcomeScreen** — toggle terms switch, tap Start
2. **InitialScreen** — tap New Wallet
3. **NewWordsScreen** — capture 24 seed words via `copyTextFrom`, tap Next
4. **BackupWords** — 5-step word validation: read position number, look up correct word via JavaScript, tap it (repeated 5 times). The component-level tests (Layer 3) additionally cover the **wrong word error path**: selecting an incorrect word shows a failure modal and navigates back.
5. **ChoosePinScreen** — enter 6-digit PIN, confirm PIN, tap Start the Wallet

The BackupWords validation uses Maestro's `runScript` command with external JavaScript files to read the step number from the accessibility tree, look up the corresponding seed word, and tap the correct option.

## Test dependencies (devDependencies only)

- `@testing-library/react-native@13.x` — component testing
- `@testing-library/jest-native@5.x` — custom Jest matchers
- `redux-saga-test-plan@4.x` — saga integration testing

No production dependencies are added. Maestro is a standalone CLI tool installed separately (`curl -Ls "https://get.maestro.mobile.dev" | bash`).

## CI integration

### Phase 1: Jest in CI (immediate)

Add to existing `.github/workflows/main.yml`:
```yaml
- name: Run tests
  run: npm test -- --ci --coverage --maxWorkers=2
```

Zero infrastructure cost — runs on existing `ubuntu-latest` runner.

### Phase 2: E2E in CI (future)

- **Android first:** Linux runner with Android emulator (`reactivecircus/android-emulator-runner`). Cost-effective.
- **iOS:** macOS runner required (10x GitHub Actions cost). Options: self-hosted Mac runner, nightly schedule only, or Maestro Cloud.

### Phase 3: Coverage thresholds

Enforce coverage thresholds in Jest config to prevent regression.

# Drawbacks
[drawbacks]: #drawbacks

1. **Production code changes for testability.** The Fabric workarounds (ToggleSwitch, Pressable migration, flex layout changes) modify production code. While the changes are minimal and the visual behavior is identical, they add components and alter layout patterns that the team must understand and maintain.

2. **Maestro is a younger tool.** The community is smaller, documentation is less comprehensive for React Native edge cases, and YAML is less expressive than TypeScript for complex test logic (e.g., the BackupWords word lookup requires external JS scripts).

3. **Black-box limitation.** Maestro cannot assert on Redux state, JS-level variables, or async operation completion. Testing wallet sync status or transaction confirmation requires waiting for UI changes rather than querying internal state.

4. **E2E tests against debug builds miss release-specific bugs.** Hermes bytecode compilation, tree-shaking, and ProGuard/R8 optimizations are not exercised. Release-build testing requires additional CI infrastructure.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Maestro (selected)

Maestro is a black-box E2E testing framework that interacts with the app through the OS accessibility layer (XCUITest on iOS, UIAutomator on Android). It never touches the JavaScript environment.

**What makes it the right choice for this project:**

- **LavaMoat/SES compatibility.** The app uses `@lavamoat/react-native-lockdown` to apply SES (Secure EcmaScript) lockdown, freezing JavaScript globals to prevent prototype pollution. Maestro operates entirely outside the app runtime, so there is zero risk of conflict with SES — now or after future updates to either library.
- **Zero npm footprint.** Maestro is a standalone CLI tool, not an npm package. It adds no dependencies to the project, no entries to `lavamoat.allowScripts`, and no native modules to compile.
- **YAML accessibility.** Test flows are written in declarative YAML, readable by QA team members without TypeScript knowledge.
- **Cross-platform.** The same YAML flow file works on both iOS and Android simulators.

**Weaknesses to be aware of:**

- No integrated build step — requires a separately built and running app.
- Black-box only — cannot assert on Redux state or JS-level behavior.
- YAML is less expressive than a full programming language for complex logic.
- Younger ecosystem with less React Native-specific documentation.

## Detox (evaluated, viable alternative)

Detox is a gray-box E2E testing framework purpose-built for React Native. It injects instrumentation into the JavaScript runtime to synchronize with the RN bridge and detect idle state.

**What it gets right:**

- **TypeScript tests.** Tests are written in the same language as the app, using familiar Jest patterns. Complex logic (like BackupWords word lookup) is native JavaScript, no external scripts needed.
- **Integrated build step.** `detox build` wraps `xcodebuild`/`gradle`, enabling a single command to build and test release binaries in CI.
- **Gray-box sync.** When working, Detox's auto-synchronization eliminates manual timeouts by waiting for the app's JS thread to become idle before each assertion.
- **Proven at scale.** Large React Native apps (Wix, Callstack clients) use Detox in production CI pipelines.

Detox was evaluated hands-on and **works with the same Fabric workarounds** described in this RFC (branch: `experiment/detox-with-fixes`, 3 tests passing through WelcomeScreen → InitialScreen → BackupWords).

**Why it was not selected:**

- **LavaMoat/SES risk.** Detox injects runtime instrumentation to synchronize with the RN bridge. This instrumentation modifies the JavaScript environment, which could conflict with SES lockdown in unpredictable ways. The conflict may not manifest immediately but creates ongoing maintenance risk as either Detox or LavaMoat updates. For a wallet app where SES is a security requirement, this is an unacceptable risk.
- **Auto-sync is non-functional.** The app has recurring 1-second JS timers (Unleash feature flag polling, WebSocket keepalive) that prevent Detox's sync mechanism from ever detecting an idle state. This forces `detoxEnableSynchronization: 0`, negating Detox's primary advantage and making it effectively a black-box tool with extra complexity.
- **Heavy npm dependency.** Detox adds `detox`, `jest-circus`, and transitive native dependencies that must be maintained, updated, audited, and added to `lavamoat.allowScripts`.

**When to reconsider Detox:**

If future requirements demand gray-box capabilities (e.g., asserting on Redux state during E2E, testing async wallet sync internals), switching to Detox would require minimal effort since the same app-level workarounds support both frameworks. The Detox branch is preserved for this purpose.

## Appium (not evaluated hands-on)

Appium's setup complexity and infrastructure overhead don't justify its benefits for a dedicated React Native project. It's designed for cross-platform testing organizations with dedicated infrastructure teams, not single-app teams.

## Impact of not doing this

Without automated tests, every feature addition, dependency update, and refactoring carries the risk of undetected regressions. QA remains the only quality gate, scaling linearly with feature complexity. Developer velocity decreases as the codebase grows because manual verification becomes the bottleneck.

# Prior art
[prior-art]: #prior-art

- **Meta's React Native team** uses Maestro for E2E testing of the framework itself, validating compatibility with both old and new architecture.
- **Expo** provides official Maestro Cloud integration in their CI workflow.
- **hathor-wallet-lib** has an integration test helper (RFC in progress) that provides race-condition-free wallet generation for tests. The patterns established there (deterministic test setup, isolated test state) inform this RFC's approach to test isolation.
- The React Native community has broadly converged on the **Jest + RNTL + Maestro** stack for 2025-2026, replacing the older **Jest + Enzyme + Detox** stack.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Release build testing in CI.** Should E2E tests run against debug or release builds? Release builds catch more production-relevant bugs but require longer build times and more CI infrastructure. The PoC uses debug builds.

2. **Android E2E.** The PoC was developed and tested on iOS only. The same Maestro YAML flows should work on Android, but this has not been validated. Android-specific workarounds may be needed.

3. **Coverage thresholds.** What coverage percentages should gate PRs? Too high creates friction; too low provides false confidence. Proposed starting point: 25% overall, 90% on financial utilities.

4. **Flakiness management.** E2E tests are inherently more flaky than unit tests. How many retries before a test is considered failing? Should flaky tests block merges?

# Future possibilities
[future-possibilities]: #future-possibilities

1. **Send transaction flow.** The next E2E flow to implement after wallet creation. Covers: Dashboard → Address → Amount → Confirm → PIN → Success.

2. **Wallet restore flow.** Import wallet via seed words — validates the LoadWordsScreen and word validation logic.

3. **Visual regression testing.** Tools like Percy or Applitools can screenshot-compare screens across builds, catching layout and styling regressions that functional tests miss.

4. **Real device testing.** Maestro Cloud and BrowserStack support running tests on real device farms, catching hardware-specific bugs that simulators miss.

5. **Performance benchmarking.** Measure app startup time, screen transition time, and wallet sync time in CI to catch performance regressions.

6. **Test-driven development workflow.** Once the test infrastructure is mature, new features can be developed test-first — writing the E2E flow and component tests before the implementation.

7. **AI agent testing guidelines via CLAUDE.md.** A dedicated section should be added to the repository's `CLAUDE.md` (or equivalent agent configuration) that establishes testing expectations for any AI agent working on the codebase. This section should be concise — a few rules with pointers — not a full testing manual:

   - **What to test:** every new feature must include unit tests for logic, component tests for UI, and an E2E flow update if the feature touches a critical user journey.
   - **How to test:** point to a dedicated knowledge file (e.g., `docs/testing-guide.md`) or a skill file that covers test patterns, helper utilities (`renderWithProviders`, `createTestStore`), mock conventions, and Maestro flow structure.
   - **When to adapt existing tests:** if modifying a screen or component that has existing tests, update or rewrite the tests to reflect the new behavior. Never delete tests without a replacement that covers the same scenarios.
   - **When to remove tests:** only when the tested feature is removed entirely. Document the removal reason in the commit message.

   The goal is that any agent — human or AI — touching the repository has clear, discoverable guidance on the project's testing standards without duplicating that guidance across multiple configuration files. The CLAUDE.md section stays brief and authoritative; the knowledge/skill file goes deep.
