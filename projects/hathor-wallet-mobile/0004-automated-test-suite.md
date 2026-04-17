- Feature Name: automated-test-suite
- Start Date: 2026-04-17
- RFC PR: (leave this empty)
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

The app requires several changes to work with E2E automation tools on React Native 0.77 + New Architecture (Fabric). These changes affect both Detox and Maestro equally — they stem from how Fabric handles the accessibility tree and touch event delivery via XCUITest.

### 1. Custom ToggleSwitch component

**Problem:** React Native's native `<Switch>` (UISwitch on iOS) does not fire `onValueChange` when tapped programmatically by XCUITest-based tools under Fabric. The native control receives the tap but Fabric's event pipeline does not dispatch the value change to the JS layer.

**Solution:** A custom `ToggleSwitch` component built with `Pressable` that visually mimics the native switch. Props are API-compatible with the native Switch (`value`, `onValueChange`, `trackColor`, `testID`).

### 2. NewHathorButton: TouchableOpacity to Pressable

**Problem:** `TouchableOpacity` (from `react-native`) does not respond to XCUITest taps in Fabric mode. The component is found in the accessibility tree but `onPress` never fires.

**Solution:** Replace with `Pressable`. Additionally, the native `disabled` prop on Pressable prevents XCUITest from registering taps even after the prop changes from `true` to `false`. The disabled behavior is moved to a JS-level guard in `onPress`.

### 3. Flex container layout fix

**Problem:** Interactive elements (buttons) placed inside a `View` with `flex: 1, justifyContent: 'flex-end'` are unreachable by XCUITest. The empty flex space above the button intercepts the tap.

**Solution:** Move buttons outside the flex container as siblings. Use `<View style={buttonView} />` as an empty spacer, with buttons rendered after it. The visual layout is identical.

### 4. testID props

`testID` props added to: terms agreement switch, Start button, seed words data element, backup step number display, and NumPad buttons. These enable both Detox and Maestro to target elements by stable identifier instead of fragile text matching.

## E2E framework: Maestro

### Why Maestro over Detox

Both frameworks were evaluated hands-on. Both work with the Fabric workarounds above. The deciding factor is **LavaMoat/SES compatibility**:

- **Detox** is a gray-box framework that injects instrumentation into the JavaScript runtime to synchronize with the RN bridge and detect idle state. This instrumentation could conflict with `@lavamoat/react-native-lockdown` which applies SES to freeze JavaScript globals. The conflict may not manifest immediately but creates ongoing maintenance risk as either library updates.

- **Maestro** is a black-box framework that operates entirely outside the app runtime via the OS accessibility layer (XCUITest on iOS, UIAutomator on Android). It never touches the JavaScript environment, making it **inherently compatible with SES lockdown**.

Additionally, Detox's auto-sync advantage is nullified in this app because recurring 1-second JS timers (Unleash polling, WebSocket keepalive) prevent the sync mechanism from ever detecting an idle state, requiring `detoxEnableSynchronization: 0`.

| Criterion | Detox | Maestro |
|---|---|---|
| LavaMoat/SES risk | Runtime instrumentation may conflict | Zero risk — fully external |
| npm dependency impact | Heavy (detox + jest-circus + native deps) | None (standalone CLI) |
| Auto-sync | Broken by recurring timers | N/A — black-box |
| Test language | TypeScript | YAML + JS scripts |
| QA accessibility | Developers only | Anyone who reads YAML |
| Build integration | Built-in `detox build` | Separate build step |

### Wallet creation flow

The PoC implements the complete wallet creation journey:

1. **WelcomeScreen** — toggle terms switch, tap Start
2. **InitialScreen** — tap New Wallet
3. **NewWordsScreen** — capture 24 seed words via `copyTextFrom`, tap Next
4. **BackupWords** — 5-step word validation: read position number, look up correct word via JavaScript, tap it (repeated 5 times)
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

2. **Maestro is a younger tool than Detox.** The community is smaller, documentation is less comprehensive for React Native edge cases, and YAML is less expressive than TypeScript for complex test logic (e.g., the BackupWords word lookup requires external JS scripts).

3. **Black-box limitation.** Maestro cannot assert on Redux state, JS-level variables, or async operation completion. Testing wallet sync status or transaction confirmation requires waiting for UI changes rather than querying internal state.

4. **E2E tests against debug builds miss release-specific bugs.** Hermes bytecode compilation, tree-shaking, and ProGuard/R8 optimizations are not exercised. Release-build testing requires additional CI infrastructure.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Why Maestro over Detox?

Detox was the initial choice and was evaluated first. It works with the same Fabric workarounds (proven on branch `experiment/detox-with-fixes`). However, its gray-box instrumentation creates a compatibility risk with LavaMoat's SES lockdown that Maestro's black-box approach eliminates entirely. Since Detox's primary advantage (auto-sync) is also non-functional in this app due to recurring timers, the trade-off favors Maestro.

### Why not Appium?

Appium's setup complexity and infrastructure overhead don't justify its benefits for a dedicated React Native project. It's designed for cross-platform testing organizations, not single-app teams.

### What is the impact of not doing this?

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
