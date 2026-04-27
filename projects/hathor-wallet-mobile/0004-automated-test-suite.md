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

The app requires three classes of changes to work with E2E automation tools on React Native 0.77 + New Architecture (Fabric):

1. A canonical accessibility-tree pattern (`accessible={false}` on layout Views) that resolves the broadest class of XCUITest tap-delivery failures **without altering layout**.
2. A custom `ToggleSwitch` component, replacing the native `<Switch>`, which has a Fabric-specific bug that prevents `onValueChange` from firing.
3. Stable `testID` props on key interactive elements.

These are **framework-agnostic** — the same pattern is required by any XCUITest-based tool (Maestro, Detox, Appium).

### 1. Canonical pattern: `accessible={false}` on layout Views

**Problem.** On iOS, when a non-interactive `View` (used purely for layout — flex spacing, rows, wrappers) does not opt out of accessibility, VoiceOver/XCUITest aggregates all of its descendants' accessibility text into a single accessibility node attached to that View. The View becomes a "monolithic" tappable region that intercepts taps meant for the interactive children inside it. Symptoms: a button inside a flex container appears in the accessibility tree, but tapping it does nothing — the parent View "swallows" the event.

**Solution (community standard).** Mark every non-interactive layout View with `accessible={false}`. This tells iOS: this is a layout box, not a tappable region. Children retain their own accessibility identity and become directly targetable.

This is the canonical fix prescribed by both Maestro's official React Native guide and React Native's own accessibility documentation:

> *"You can resolve these issues by enabling accessibility for the inner component and disabling it for the outer container."* — [Maestro React Native docs](https://docs.maestro.dev/get-started/supported-platform/react-native)

> *"VoiceOver disallow[s] nested accessibility elements... accessibility focus is only available on the first child view with the `accessible` property, and not for the parent or sibling without `accessible`."* — [React Native Accessibility docs](https://reactnative.dev/docs/accessibility)

Related upstream issues:
- [facebook/react-native#24515](https://github.com/facebook/react-native/issues/24515) — Nested accessibility items are not respected on iOS
- [mobile-dev-inc/Maestro#3056](https://github.com/mobile-dev-inc/Maestro/issues/3056) — RN iOS elements disappear from accessibility tree

**The rule of thumb (lift this verbatim into CLAUDE.md / a skill file):**

> On iOS React Native, every non-interactive `View` used for layout (flex spacers, rows, wrappers, decorative containers) **must** have `accessible={false}`. Interactive elements (`Pressable`, `TouchableOpacity`, `TouchableHighlight`, `Switch`, custom controls) inherit `accessible={true}` from their role and need no annotation. Container Views that intentionally group content for screen readers (e.g., a card with a single combined label) get `accessible={true}` plus an explicit `accessibilityLabel`.
>
> This is good a11y hygiene independent of testing — it prevents VoiceOver from announcing redundant layout boxes and keeps the accessibility tree flat.

**Why this replaces earlier workarounds.** A previous iteration of this PoC moved buttons out of `flex: 1` containers to make them reachable by XCUITest. That was a layout-level workaround for a problem that the accessibility tree exposes. The canonical pattern keeps the original layout intact — buttons stay inside their `buttonView` flex container — and resolves the issue at the accessibility layer where it belongs. Production layout choices should not be driven by E2E framework quirks when the framework itself documents an escape hatch.

**A previous iteration also migrated `NewHathorButton` from `TouchableOpacity` to `Pressable` with a JS-level disabled guard.** That migration was a workaround for symptoms caused by the same accessibility-tree aggregation. With `accessible={false}` applied correctly to the parent layout Views, `TouchableOpacity` and the native `disabled` prop work as expected. The button has been kept on `TouchableOpacity` accordingly.

### 2. Custom ToggleSwitch component

**Problem.** React Native's native `<Switch>` (UISwitch on iOS) does not fire `onValueChange` when tapped programmatically by XCUITest-based tools under Fabric. The native control receives the tap but Fabric's event pipeline does not dispatch the value change to the JS layer. This is independent of the accessibility-tree issue above. Related issues:
- [facebook/react-native#43648](https://github.com/facebook/react-native/issues/43648) — Missing accessibilityLabel on iOS for Switch in Fabric
- [wix/Detox#4803](https://github.com/wix/Detox/issues/4803) — isReady timeout with RN 0.76+

**Solution.** A small `ToggleSwitch` component built with `Pressable` that visually mimics the native switch. Props are API-compatible with the native Switch (`value`, `onValueChange`, `trackColor`, `testID`). Inner decorative `View`s (track, thumb) carry `accessible={false}`, so only the outer `Pressable` exposes accessibility identity.

### 3. testID props

`testID` props added to: terms agreement switch, Start button, seed-words data element, backup step number display, and NumPad buttons. These enable E2E tools to target elements by stable identifier instead of fragile text matching that breaks when copy is changed or localized.

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

1. **Minor production code changes for testability.** Three classes of changes are added: `accessible={false}` on layout Views, a custom `ToggleSwitch` replacing the native one, and `testID` props on key interactive elements. The first is defensible as a11y hygiene independent of testing; the second is required to work around a Fabric/UISwitch bug regardless of which test framework is used; the third is a stable identifier with no UX impact. Crucially, **no layout patterns are altered** for testability — the original component tree and flex hierarchy are preserved.

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

5. **End-to-end re-verification of the canonical accessibility pattern.** The `accessible={false}` pattern was applied after the original PoC's layout-based workaround (and its accompanying `Pressable` migration) had already proven the wallet-creation flow could pass under Maestro. The pattern is the documented standard prescribed by Maestro itself, and all Jest/component tests pass with the new code. A full Maestro run of `walletCreation.yaml` against the canonical-pattern code is the remaining verification step before this PoC is shipped to CI.

# Future possibilities
[future-possibilities]: #future-possibilities

1. **Send transaction flow.** The next E2E flow to implement after wallet creation. Covers: Dashboard → Address → Amount → Confirm → PIN → Success.

2. **Wallet restore flow.** Import wallet via seed words — validates the LoadWordsScreen and word validation logic.

3. **Visual regression testing.** Tools like Percy or Applitools can screenshot-compare screens across builds, catching layout and styling regressions that functional tests miss.

4. **Real device testing.** Maestro Cloud and BrowserStack support running tests on real device farms, catching hardware-specific bugs that simulators miss.

5. **Performance benchmarking.** Measure app startup time, screen transition time, and wallet sync time in CI to catch performance regressions.

6. **Test-driven development workflow.** Once the test infrastructure is mature, new features can be developed test-first — writing the E2E flow and component tests before the implementation.

7. **AI agent testing guidelines via CLAUDE.md and a dedicated skill file.** A dedicated section in `CLAUDE.md` should establish testing expectations for any AI agent working on the codebase, paired with a deeper skill/knowledge file that the agent loads on demand. The structure follows the progressive-disclosure pattern recommended for AI agent configuration: brief authoritative rules in `CLAUDE.md`, depth in a separate file.

   ### `CLAUDE.md` section (brief, authoritative)

   This is the rule set the agent always sees. Keep it under ~30 lines:

   - **TDD by default.** Every new feature includes: unit tests for logic, component tests for UI screens, and an E2E flow update if the feature touches a critical user journey (wallet creation, send/receive, backup/restore, PIN handling).
   - **Adapting existing tests.** When modifying a screen or component that has existing tests, update or rewrite those tests to match the new behavior. Never delete a test without a replacement that covers the same scenarios.
   - **Removing tests.** Only when the tested feature is removed. Document the removal reason in the commit message.
   - **Testability boundary.** Production code changes for testability are limited to: (a) `accessible={false}` on non-interactive layout Views, (b) `testID` props, (c) the existing `ToggleSwitch` component. **Do not** introduce layout changes, replace `TouchableOpacity` with `Pressable`, or move components in the component tree to make E2E tests work — those are symptoms of accessibility-tree issues, not problems to solve at the layout level.
   - **For deeper testing guidance:** load the testing skill file (e.g., `docs/testing-guide.md` or `.claude/skills/wallet-testing.md`).

   ### Skill / knowledge file content (depth, loaded on demand)

   The companion file goes deep on patterns and is structured so an agent can find specific guidance fast. Recommended sections:

   1. **Test pyramid layout** — file paths, what each layer covers, how to run each (`npm test`, `npm test -- --testPathPattern=...`, `maestro test ...`).
   2. **Test patterns** — `renderWithProviders`, `createTestStore`, mock conventions for `@hathor/wallet-lib`, navigation mocks, image asset mocks.
   3. **The iOS accessibility-tree rule** — the `accessible={false}` pattern, with the exact wording from the rule of thumb above. This is the single most common mistake an agent will make: writing a new screen with nested layout Views and then being unable to E2E-test it. The skill must surface this rule prominently.
   4. **The Fabric workarounds catalogue** — the existing `ToggleSwitch` exists because of [facebook/react-native#43648](https://github.com/facebook/react-native/issues/43648); document it so an agent doesn't try to "modernize" it back to the native `<Switch>`.
   5. **Maestro flow conventions** — how to structure `.maestro/flows/*.yaml` files, when to use `runScript` vs inline commands, how to capture and reuse data across steps (`copyTextFrom` + `output.X`), and the cleanup discipline (`.maestro/cleanup.sh` to manage simulator memory pressure).
   6. **What NOT to test in E2E** — release-build behavior, real network conditions, OS permissions, visual regressions. These are explicitly out of scope for Maestro and belong to QA or other tooling.

   ### Why this matters

   AI agents working on this codebase will frequently add new screens, modify existing ones, or refactor shared components. Without this guidance, a well-intentioned agent will: (a) skip writing tests because no one told it that's required, (b) write tests that pass locally but fail in E2E because new layout Views weren't marked `accessible={false}`, (c) "fix" Fabric workarounds it perceives as quirky, breaking E2E. The CLAUDE.md + skill file combination prevents all three failure modes by making the rules discoverable at the right level of detail for the task at hand.
