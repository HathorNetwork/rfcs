- Feature Name: Feature Activation Phased Testing
- Start Date: 2023-08-01
- Initial Document: [Feature Activation](https://docs.google.com/document/d/1IiFTVW1wH6ztSP_MnObYIucinOYd-MsJThprmsCIdDE/edit)
- Author: Gabriel Levcovitz <<gabriel@hathor.network>>

# Summary
[summary]: #summary

This RFC describes a phased procedure for testing the Feature Activation project.

# Motivation
[motivation]: #motivation

After all changes introduced by the previous Feature Activation RFCs are implemented, the mechanism should be tested with some "dull" features in both the testnet and mainnet environments. Such features do not change any full node behavior and are used exclusively to test the Feature Activation mechanism itself. Only then we'll be able to use it with confidence in the mainnet to change real features like updating the maximum merkle path length, for example.

# Guide-level explanation
[Guide-level explanation]: #guide-level-explanation

The testing will consist of 3 phases:

1. NOP features in testnet
2. Maximum merkle path length update in testnet
3. NOP features in mainnet

Phase 1 will happen independently, and then Phase 2 and Phase 3 can happen concurrently. Note: updating the maximum merkle path length in mainnet is outside the scope of this project and will be described in a separate RFC.

The NOP features used in Phases 1 and 3 are simply defined in the respective network's settings file, but their activation has no real effect in any computation of the full node, that is, no forks are created no matter whether those features are activated or not. Instead, their states will simply be logged by the full node.

After Phase 1 is complete, Phase 2 will introduce a real change that creates a fork in testnet between nodes that recognize its feature activation, and nodes that do not. At the same time, Phase 3 can start. It is analogous to Phase 1, but on mainnet instead of testnet.

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

In this section, technical details are expanded for what was described above.

## Phase 1

In this phase, 3 NOP features are defined for testnet and each one of them undergo their own feature activation process. Then, we validate that their states progress as expected, both by inspecting their values through the Explorer interfaces, and by inspecting logs via AWS CloudWatch queries.

### NOP features configuration

We'll be using 3 NOP features and define them in `testnet.yml`. We'll test different Feature Activation behavior by configuring different settings for each feature:

1. `NOP_FEATURE_1` is expected to be signaled and activated through bit signaling. Therefore, its `signal_support_by_default` will be set to `True`, so testnet miners (the ones used in `tx-mining-service`) will receive block templates with their signal bit enabled. It'll also have a `minimum_activation_height`.
2. `NOP_FEATURE_2` is expected to NOT be activated through bit signaling. Instead, it'll have `lock_in_on_timeout=True` and will be "forcefully" activated.
3. `NOP_FEATURE_3` is expected to FAIL. It'll have `signal_support_by_default=False` and `lock_in_on_timeout=False`.

By doing this, we'll test different paths of the Feature Activation workflow. To define the configuration, we'll first choose some block height `N`, that is, the block height where Phase 1 will start. It can only be chosen when we actually implement this RFC. All other heights are relative to `N`. Here's the reference configuration:

```yml
# testnet.yml

FEATURE_ACTIVATION:
  default_threshold: 30240 # 30240 = 75% of evaluation_interval (40320)
  features:
    NOP_FEATURE_1:
      bit: 0
      start_height: <N>
      timeout_height: <N + 2 * 40320> # 4 weeks after the start
      minimum_activation_height: <N + 3 * 40320> # 6 weeks after the start
      lock_in_on_timeout: false
      version: <insert the hathor-core version>
      signal_support_by_default: true

    NOP_FEATURE_2:
      bit: 1
      start_height: <N>
      timeout_height: <N + 2 * 40320> # 4 weeks after the start
      minimum_activation_height: 0
      lock_in_on_timeout: true
      version: <insert the hathor-core version>
      signal_support_by_default: false

    NOP_FEATURE_3:
      bit: 2
      start_height: <N>
      timeout_height: <N + 2 * 40320> # 4 weeks after the start
      minimum_activation_height: 0
      lock_in_on_timeout: false
      version: <insert the hathor-core version>
      signal_support_by_default: false
```

And below are the expected states for each feature for each height, according to the [state transitions](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#state-transitions) workflow defined in the original RFC. Note that `40320` is the evaluation interval, which is equivalent to 2 weeks.

#### `NOP_FEATURE_1`

1. It begins as `DEFINED`, up to block `N - 1`.
2. It transitions to `STARTED` on block `N`, for reaching its `start_height`.
3. After 2 weeks, it gets to `LOCKED_IN` on block `N + 40320`, for reaching its bit signaling `threshold` before the `timeout_height`.
4. After 2 weeks, it continues `LOCKED_IN` on block  `N + 2 * 40320`, as its `minimum_activation_height` has not yet been reached.
5. After 2 weeks, it gets to `ACTIVE` on block `N + 3 * 40320`, as its `minimum_activation_height` is reached.

Therefore, `NOP_FEATURE_1`'s process takes 6 weeks.

#### `NOP_FEATURE_2`

1. It begins as `DEFINED`, up to block `N - 1`.
2. It transitions to `STARTED` on block `N`, for reaching its `start_height`.
3. After 2 weeks, it gets to `MUST_SIGNAL` on block `N + 40320`, for its `lock_in_on_timeout` configuration.
4. After 2 weeks, it gets to `LOCKED_IN` on block  `N + 2 * 40320`.
5. After 2 weeks, it gets to `ACTIVE` on block `N + 3 * 40320`.

Therefore, `NOP_FEATURE_2`'s process takes 6 weeks.

#### `NOP_FEATURE_3`

1. It begins as `DEFINED`, up to block `N - 1`.
2. It transitions to `STARTED` on block `N`, for reaching its `start_height`.
3. After 2 weeks, it continues on `STARTED` on block `N + 40320`, as its `timeout_height` has not been reached yet.
4. After 2 weeks, it gets to `FAILED` on block  `N + 2 * 40320`, as its `timeout_height` is reached and `lock_in_on_timeout` is `false`.

Therefore, `NOP_FEATURE_3`'s process takes 4 weeks.


### Validation via the Hathor Explorer

By manually inspecting blocks via the Hathor Explorer interfaces described in the [Feature Activation Explorer UIs RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0003-explorer-uis.md), we'll be able to validate the expected state for each block-feature pair according to the lists described above.

### Validation via logging

Complementing the validation via Explorer, we'll also have validation via full node logs, which will be useful to perform more complex queries to understand state transitions, specially if debugging is necessary. By using structured logging of features states for each block, we'll be able to leverage AWS CloudWatch queries to perform analysis and even export metrics.

After a new block is received and processed by the full node, we'll log its height, together with its state for each NOP feature. At the end of the `HathorManager.tx_fully_validated()` method, we'll call a new `HathorManager.log_feature_states()` method, defined as follows:

```python
def log_feature_states(self, vertex: BaseTransaction) -> None:
    if not isinstance(vertex, Block):
        return

    feature_descriptions = self.feature_service.get_bits_description(block=vertex)
    state_by_feature = {
        feature.value: description.state.value
        for feature, description in feature_descriptions.items()
    }

    self.log.info(
        'New block accepted with feature activation states',
        block_height=vertex.get_height(),
        features_states=state_by_feature
    )
```

This logging will be removed after Phase 1 is complete.

## Phase 2

Phase 2 can start as soon as Phase 1 ends. Let `M` be a block height for the beginning of Phase 2. It'll define a new `INCREASE_MAX_MERKLE_PATH_LENGTH` feature that will actually affect how the full node verifies merged mining blocks.

Currently, there's a `MAX_MERKLE_PATH_LENGTH: int = 12` constant in `aux_pow.py`. This constant will be replaced by a function call to `get_max_merkle_path_length()`, with its reference implementation as follows:

```python
def get_max_merkle_path_length(feature_service: FeatureService, block: Block) -> int:
    if feature_service.is_feature_active(
      block=block,
      feature=Feature.INCREASE_MAX_MERKLE_PATH_LENGTH
    ):
        return NEW_MAX_MERKLE_PATH_LENGTH

    return OLD_MAX_MERKLE_PATH_LENGTH
```

Where this function is actually defined will be left as an implementation detail. TODO: what's a good value for `NEW_MAX_MERKLE_PATH_LENGTH`?

Then, its reference configuration is:

```yml
# testnet.yml

FEATURE_ACTIVATION:
  default_threshold: 30240 # 30240 = 75% of evaluation_interval (40320)
  features:
    INCREASE_MAX_MERKLE_PATH_LENGTH:
      bit: 3
      start_height: <M>
      timeout_height: <M + 2 * 40320>
      minimum_activation_height: 0
      lock_in_on_timeout: false
      version: <insert the hathor-core version>
      signal_support_by_default: true
```

And it's expected to reach its threshold and become active in 4 weeks (2 in `STARTED`, 2 in `LOCKED_IN`). We'll use the same validation mechanisms described in Phase 1 to make sure the feature states transition as expected. Also, we should configure a merged miner to mine blocks with a higher merkle path length, making sure its blocks are invalidated before the feature becomes `ACTIVE`, and validated after it does.

## Phase 3

Phase 3 can start as soon as Phase 1 ends, and it is mostly analogous to Phase 1, minus a few exceptions:

- The NOP features will be configured on `mainnet.yml` instead of `testnet.yml`.
- `N` will be different.
- We won't test `NOP_FEATURE_2`, that is, we won't test `lock_in_on_timeout=True` on mainnet, as we have no easy way of coordinating bit signals with miners during the `MUST_SIGNAL` phase.

## Conclusion

As described above, Phase 1 takes 6 weeks, and Phase 2 and Phase 3 can run concurrently, respectively for 6 and 4 weeks. This makes for a total Phased Testing duration of 12 weeks. If this duration is deemed too high, we can actually shift Phase 2 and 3 earlier, as they do not conflict with Phase 1 in any way. Another option to make it even faster, is decreasing the `EVALUATION_INTERVAL`, for example from 2 weeks to 1 week, only in testnet.

After all phases are completed, we'll have enough confidence on the process' stability to use it to increase the maximum merkle path length on mainnet.

# Task breakdown

| Task                                                                                      | Dev days |
|-------------------------------------------------------------------------------------------|----------|
| [Phase 1] Release a new hathor-core version with NOP features on testnet                  | 0.5      |
| [Phase 1] Validate feature states during the following weeks                              | 0.5      |
| [Phase 2] Run a local merged miner on testnet                                             | 1        |
| [Phase 2] Release a new hathor-core version with the merkle path update on testnet        | 0.5      |
| [Phase 2] Validate feature states and merkle path verification during the following weeks | 0.5      |
| [Phase 3] Release a new hathor-core version with NOP features on mainnet                  | 0.5      |
| [Phase 3] Validate feature states during the following weeks                              | 0.5      |
| **Total**                                                                                 | **4**    |
