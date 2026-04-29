# Annex — Phase 4 (E2E Testing): Deep Dive

Companion document to [RFC 0004 — Automated Test Suite](./automated-test-suite.md).

This annex contains the in-depth technical material for **Phase 4 (Layer 4): end-to-end testing with Maestro**. It is split out of the main RFC because the implementation reality on React Native 0.77 + New Architecture (Fabric) requires several non-obvious production-code changes that warrant their own focused treatment. The main RFC summarizes Phase 4 at a high level and points here for the full reasoning, evidence, and verification.

#### Table of content

- [Production code changes](#production-code-changes)
  - [1. Accessibility tree: `accessible={false}` on layout Views](#1-accessibility-tree-accessiblefalse-on-layout-views)
  - [2. Pressable migration with JS-level disabled guard](#2-pressable-migration-with-js-level-disabled-guard)
  - [3. Flex-container layout](#3-flex-container-layout)
  - [4. Custom `ToggleSwitch` component](#4-custom-toggleswitch-component)
  - [5. testID props](#5-testid-props)
  - [Combined effect](#combined-effect)
- [Wallet creation E2E flow](#wallet-creation-e2e-flow)

## Production code changes

The app requires changes to work with E2E automation tools on React Native 0.77 + New Architecture (Fabric). **Three independent issues** must each be addressed; fixing only one or two is not enough. This was discovered the hard way: an earlier iteration applied only the doc-prescribed accessibility fix and the Maestro flow silently failed at the very first button tap. The hierarchy dump showed the button as a clean isolated accessibility node — XCUITest found it — yet the synthetic tap never reached the JS `onPress`. Two more layers of fix were needed to bridge the gap.

The full list of fixes:

1. **Accessibility tree:** `accessible={false}` on non-interactive layout Views.
2. **Pressable migration:** `Pressable` instead of `TouchableOpacity` for app buttons, with a JS-level `disabled` guard.
3. **Flex-container layout:** buttons rendered as siblings of, not inside, `flex: 1, justifyContent: 'flex-end'` containers.
4. **Custom `ToggleSwitch`** replacing the native `<Switch>` (a separate Fabric bug).
5. **`testID` props** on key interactive elements.

These are **framework-agnostic** — every XCUITest-based tool (Maestro, Detox, Appium) hits the same three iOS issues.

### 1. Accessibility tree: `accessible={false}` on layout Views

**Problem.** On iOS, when a non-interactive `View` (flex spacing, rows, wrappers) does not opt out of accessibility, VoiceOver/XCUITest aggregates all of its descendants' accessibility text into a single accessibility node attached to that View. The View becomes a "monolithic" tappable region whose `accessibilityText` reads "Welcome to Hathor Wallet! ... I agree with Terms of Service ... START" — a giant blob that swallows queries for individual children.

**Solution.** Mark every non-interactive layout View with `accessible={false}`. This tells iOS: this is a layout box, not a tappable region. Children retain their own accessibility identity and become directly queryable by `testID`.

Documented by Maestro and React Native themselves:

> *"You can resolve these issues by enabling accessibility for the inner component and disabling it for the outer container."* — [Maestro React Native docs](https://docs.maestro.dev/get-started/supported-platform/react-native)

> *"VoiceOver disallow[s] nested accessibility elements... accessibility focus is only available on the first child view with the `accessible` property, and not for the parent or sibling without `accessible`."* — [React Native Accessibility docs](https://reactnative.dev/docs/accessibility)

Related upstream issues:
- [facebook/react-native#24515](https://github.com/facebook/react-native/issues/24515) — Nested accessibility items are not respected on iOS
- [mobile-dev-inc/Maestro#3056](https://github.com/mobile-dev-inc/Maestro/issues/3056) — RN iOS elements disappear from accessibility tree

**Verification.** After this fix the hierarchy dump shows the START button as a discrete `accessibilityText: "STARTˮ` node. Without it, every parent View aggregates the entire screen's text. **But the synthetic tap still doesn't reach JS without fixes #2 and #3.** This is the trap that the docs alone don't warn about.

### 2. Pressable migration with JS-level disabled guard

**Problem.** Even with the accessibility tree clean, `TouchableOpacity.onPress` does not fire under Fabric + XCUITest. The button is found by `testID`, the synthetic tap is dispatched, and the touch is observed in iOS-level instrumentation — but Fabric's event pipeline does not deliver it as a press to the JS layer. The button visually unrelated, but its callback never runs.

**Solution.** Replace `TouchableOpacity` with `Pressable`. Pressable's gesture path is implemented differently and does deliver synthetic taps to JS. **Caveat:** Pressable's native `disabled` prop also blocks XCUITest from registering taps, even after the prop transitions from `true` to `false`. The disabled state must be implemented as a JS-level guard:

```jsx
<Pressable
  testID={props.testID}
  onPress={props.disabled ? undefined : props.onPress}
  // ...
>
```

Related upstream issues:
- [facebook/react-native#44610](https://github.com/facebook/react-native/issues/44610) — Pressable gets stuck under new architecture
- [facebook/react-native#34999](https://github.com/facebook/react-native/issues/34999) — onPress not firing occasionally
- [software-mansion/react-native-screens#2219](https://github.com/software-mansion/react-native-screens/issues/2219) — Pressable elements not working with new architecture on native-stack

### 3. Flex-container layout

**Problem.** A button placed inside a `<View style={{ flex: 1, justifyContent: 'flex-end' }}>` container is unreachable by XCUITest synthetic touches even when both the container is `accessible={false}` and the button is `Pressable`. The empty flex space above the button intercepts the synthetic tap somewhere in the gesture-handler stack, before it reaches the button's Pressable. This is a hit-test issue, not an accessibility issue.

**Solution.** Render the button as a sibling of the flex spacer, not nested inside it:

```jsx
// Before — button is inside the flex spacer, unreachable
<View accessible={false} style={style.buttonView}>
  <NewHathorButton onPress={handlePress} title="Start" />
</View>

// After — button is a sibling, spacer is empty
<NewHathorButton onPress={handlePress} title="Start" />
<View accessible={false} style={style.buttonView} />
```

The visual layout is identical (the spacer still pushes the button toward the bottom of the screen because `flex: 1, justifyContent: 'flex-end'` on the spacer combined with sibling order achieves the same effect). The hit-test difference: the button is now a peer of the spacer in the layout, not nested inside its tap-intercepting region.

### 4. Custom `ToggleSwitch` component

**Problem.** React Native's native `<Switch>` (UISwitch on iOS) does not fire `onValueChange` when tapped programmatically by XCUITest-based tools under Fabric. The native control receives the tap but Fabric's event pipeline does not dispatch the value change to the JS layer. This is independent of all three issues above. Related:
- [facebook/react-native#43648](https://github.com/facebook/react-native/issues/43648) — Missing accessibilityLabel on iOS for Switch in Fabric
- [wix/Detox#4803](https://github.com/wix/Detox/issues/4803) — isReady timeout with RN 0.76+

**Solution.** A small `ToggleSwitch` component built with `Pressable` that visually mimics the native switch. Props are API-compatible with the native Switch (`value`, `onValueChange`, `trackColor`, `testID`). Inner decorative `View`s (track, thumb) carry `accessible={false}`, so only the outer `Pressable` exposes accessibility identity.

### 5. testID props

`testID` props added to: terms agreement switch, Start button, seed-words data element, backup step number display, and NumPad buttons. These enable E2E tools to target elements by stable identifier instead of fragile text matching that breaks when copy is changed or localized.

### Combined effect

With all four code fixes applied, the full Maestro `walletCreation.yaml` flow passes 30+ steps end-to-end on iPhone 17 / iOS 26.4 / Xcode 26.4.1: WelcomeScreen → InitialScreen → NewWordsScreen → BackupWords (5-step validation) → ChoosePinScreen (PIN + confirm) → wallet started. Removing any one of fixes 1, 2, or 3 causes the very first button tap (`welcome-start-button`) to silently fail.

## Wallet creation E2E flow

The PoC implements the complete wallet creation journey:

1. **WelcomeScreen** — toggle terms switch, tap Start
2. **InitialScreen** — tap New Wallet
3. **NewWordsScreen** — capture 24 seed words via `copyTextFrom`, tap Next
4. **BackupWords** — 5-step word validation: read position number, look up correct word via JavaScript, tap it (repeated 5 times). The component-level tests (Layer 3) additionally cover the **wrong word error path**: selecting an incorrect word shows a failure modal and navigates back.
5. **ChoosePinScreen** — enter 6-digit PIN, confirm PIN, tap Start the Wallet

The BackupWords validation uses Maestro's `runScript` command with external JavaScript files to read the step number from the accessibility tree, look up the corresponding seed word, and tap the correct option.
