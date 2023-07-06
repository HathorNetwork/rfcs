- Feature Name: Feature Activation Bit Signaling
- Start Date: 2023-06-26
- Author: Gabriel Levcovitz <<gabriel@hathor.network>>

# Table of Contents

# Summary
[summary]: #summary

Given the [Feature Activation for Blocks RFC](./0001-feature-activation-for-blocks.md), a mechanism is provided to deploy new features in the Hathor network conditionally on some criteria. Part of the defined criteria is the ability for miners to signal their support for a new feature by setting signal bits on a mined block. The former RFC describes how this bit is calculated, interpreted, etc., but details on how miners would actually set those bits were left open. This RFC addresses this requirement by defining a user-friendly way for miners to configure their support for a feature, or the lack thereof.

# Motivation
[motivation]: #motivation

In the current implementation, the Feature Activation process can be used but would require manual manipulation of block bits by miners to set support for features undergoing the process. Instead, we should provide a user-friendly way through our full node CLI interface so miners can easily start their mining full node with the desired feature support configuration. That configuration will then be reflected on mining block templates, so the templates carry the corresponding signal bits for all features.

# Guide-level explanation
[Guide-level explanation]: #guide-level-explanation

Users running the full node, or specifically miners in this case, should be able to pass two new options to the `run_node` CLI interface:

1. `--signal-support [feature]`, and
2. `--signal-not-support [feature]`

Here are rule definitions for these options:

1. Both options receive the name of some feature, as defined in the existing `Feature` enum;
2. Both options can be used multiple times to either enable or disable support for multiple features;
3. A feature cannot be set in both options at the same time, as this would be a conflicting configuration;
4. A default bit signal value (either enabled or disabled) should be provided for each feature;
5. If a feature is set in any of these options while it's not undergoing a signaling phase of its Feature Activation process, the full node should emit a warning.

Here's an example of running the full node with those options, enabling support for features 2 and 3, and disabling support for feature 1:

```bash
poetry run hathor-cli run_node --status 8080 --testnet --memory-storage --signal-not-support NOP_FEATURE_1 --signal-support NOP_FEATURE_2 --signal-support NOP_FEATURE_3
```

And here's the equivalent example using env vars instead of the CLI options directly:

```bash
HATHOR_SIGNAL_SUPPORT=[NOP_FEATURE_2,NOP_FEATURE_3] HATHOR_SIGNAL_NOT_SUPPORT=NOP_FEATURE_1 poetry run hathor-cli run_node --status 8080 --testnet --memory-storage
```

Let's assume all features are disabled by default, and that `NOP_FEATURE_1` is configured to use `bit=0`, `NOP_FEATURE_2` is configured to use `bit=1`, and `NOP_FEATURE_3` is configured to use `bit=2`, where `bit=0` is the LSB, as explained in the previous RFC.

Then, the signal bits for this full node run would be `110`, meaning that those bits would be present in all block templates generated for mining.

The full node should log the outcome for each feature, indicating whether its support is enabled or disabled, and whether the reason was the default value or the user setting.

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

## Bit Signaling Service

A new `BitSignalingService` class should be implemented, providing a single public method: `get_signal_bits()`, that returns an `int`. This class and method's only purpose is to calculate the binary number that represents the signal bits for all block templates generated during this full node run, as configured by the respective CLI options and feature configuration.

That class should also perform validations on the configuration, as specified in the list in the [Guide-level explanation] section. The full node must not start if those validations fail.

## New signal_support_by_default configuration

A new attribute called `signal_support_by_default` should be implemented in the `Criteria` class. It is a boolean that represents whether a feature should be supported by default or not, according to its configuration via `HathorSettings`. Its default value should be `False`.

That is, assuming `signal_support_by_default=True` for `NOP_FEATURE_1`, and no CLI option is provided for that feature, then that feature is going to be signaled as supported in block templates. If some CLI option is set for that feature, then it takes precedence over the `signal_support_by_default` attribute.

## Hathor Manager and mining APIs

There are changes to be made on `HathorManager` and in mining APIs, be it over WebSocket or HTTP. The manager should call the `BitSignalingService.get_signal_bits()` method and generate template blocks accordingly. The specifics of where those changes should be are left as unresolved implementation details, but should be made so that all mining APIs return the signal bits as generated by the new service, that is, all block template generators must also be updated accordingly.

# Proof of Concept

A POC is provided for this RFC, as a [pull request on hathor-core](https://github.com/HathorNetwork/hathor-core/pull/688).
