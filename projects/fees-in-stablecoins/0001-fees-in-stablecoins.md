- Feature Name: fees-in-stablecoins
- Start Date: 2026-07-24
- Initial Document: [Pay fees in stablecoins](https://docs.google.com/document/d/1hcFsHTEFO-t9aoV6TCRSdQgCpV9mUAFxims0lHZpiyI/edit?tab=t.0)
- Author: Gabriel Levcovitz <gabriel@hathor.network>

# Summary
[summary]: #summary

This document proposes allowing users to pay fees in stablecoins. Currently, fees can only be paid in HTR or deposit-based tokens.

# Motivation
[motivation]: #motivation

Fee payment is one of the highest friction points in Web3. Most networks accept fees only in their native token, which becomes a significant limitation as stablecoins and other real-world asset (RWA) tokens gain adoption. Some RWA-focused chains already allow fees to be paid directly in stablecoins (e.g., Tempo, Stable). Hathor should offer the same capability in order to remain competitive in this market.

This raises the question of what constitutes a stablecoin: the protocol should define a fixed set of supported tokens, together with the fee values charged when each one is used. For illustration, a USD stablecoin would pay \$1 in fees, while a BRL stablecoin would pay R\$5. These amounts are not expected to track real-world prices; they may be adjusted in discrete protocol upgrades.

The protocol currently charges the following fees:

- Fee-based tokens: 0.01 HTR per output;
- Privacy fees: 0.01 or 0.02 HTR per output, depending on the shielding mode;
- Nano fees: still under discussion.

The proposal is therefore to define distinct fee values for each token accepted as a supported stablecoin. Furthermore, unlike the existing fees, which are burned, stablecoin fees should be deposited into an address controlled by Hathor Labs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Requirements

We begin by enumerating the business decisions derived from the initial project document linked above:

- **B1 (Business decision 1):** Allow fees to be paid with tokens other than HTR and deposit-based tokens.
- **B2:** Fees paid in stablecoins should not be burned; they should be sent to a designated address.
- **B3:** There should be a configurable list determining which tokens may be used to pay fees, along with their corresponding fee values.
- **B4:** Setting Nano fees aside, we consider three fee types: fee-based tokens, amount-shielded privacy (amount only), and fully-shielded privacy (amount and token).
- **B5:** Each token may define different fee values for each fee type.
- **B6:** Each token may have a different destination address for its fees, called a deposit address.
- **B7:** We should be able to update every configuration (which tokens are accepted, the fee values, and the deposit addresses).
- **B8:** Add hUSDC as an accepted fee token, with the following values:
  - Fee-based tokens: 0.005 hUSDC per output;
  - Privacy fees: 0.01 (amount only) or 0.02 (amount and token) hUSDC per output.
- **B9:** Update the fees paid in HTR (and consequently, in deposit-based tokens):
  - Fee-based tokens: 1 HTR per output;
  - Privacy fees: 2 (amount only) or 4 (amount and token) HTR per output.
- **B10:** Users may not mix stablecoin fees with HTR or deposit-based token fees in the same transaction, nor may they mix two different stablecoins. The possible cases are therefore:
  - Pay fees with a combination of HTR and deposit-based tokens (already supported).
  - Pay fees in a single stablecoin;

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section details the low-level implementation.

## Pre-refactors

This section describes a few refactors that must be implemented before the new features.

### Fee configuration

To support the new configurable format with multiple fee tokens (**B3**), we must refactor the way fees are hardcoded in the protocol. Currently, a single `FEE_PER_OUTPUT_V1` constant in `HathorSettings` defines the amount of HTR, in Token Amount V1, that each output interacting with a fee-based token must account for (or, analogously, each Nano Contract action):

```python
# Fee rate setting in V1 decimals (that is, 0.01 HTR)
FEE_PER_OUTPUT_V1: int = 1
```

As a concrete example, a transaction with three fee-based token outputs multiplies this value by three, yielding the fee as an HTR amount in Token Amount V1 (that is, with two decimal places), so the effective fee is 0.03 HTR.

This structure is insufficient for the new requirements: it supports neither multiple tokens (**B3**), nor distinct fee values per token (**B5**), nor sub-cent values (**B8**), and it does not cover the fees charged for privacy operations (**B4**). We must introduce a new configuration model for fees and thread it across the codebase. This can be implemented as a standalone, pure-refactor PR covering only HTR, before the introduction of new features (more tokens, the new values, deposit addresses, and so on). No behavior should change at this step — only the configuration representation.

This is the new settings model:

```yaml
FEE_POLICIES:
  v1:
    # HTR
    "00":
      fee_based_tokens: "0.01"
      amount_shielded: "0.01"
      full_shielded: "0.02"
```

Fee policies are versioned to satisfy **B7**. Version V1 is mandatory and represents the current state of the protocol. Each policy is a dictionary in which each key is the UID of a token that may be used to pay fees (**B3**, `00` is HTR), and each value is a struct defining the amount charged for every fee type (**B5**), in decimal string notation (convertible to versioned amounts through `UnsignedAmount.parse`). This replaces the `FEE_PER_OUTPUT_V1` setting and the `FEE_TOKEN_AMOUNT_PER_OUTPUT` computed property derived from it. We must enforce that V1 is defined for all networks and that HTR is defined in all policies.

<!--
This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.
-->

### Balance Accounting

Fees are charged in two contexts: UTXO balance verification and Nano actions. A pre-refactor is also warranted to unify these charges, which are currently duplicated.

During UTXO balance verification, while constructing the `TokenInfoDict`, we calculate the total fee represented in the `FeeHeader` through `FeeHeader.total_fee_amount`. This method sums all fees paid in HTR with all fees paid in deposit-based tokens, applying the appropriate constant conversion between them (100 deposit-based tokens are equivalent to 1 HTR). The sum is stored in the `TokenInfoDict` for later use in the verification method, where it is checked against the expected value calculated from the count of fee-related outputs. During construction, we also check whether the token may be used to pay fees and, crucially, subtract the fee values from the input/output balance. In other words, we create an imbalance in the transaction that accounts for the fees being burned.

Analogously, the Nano Contracts Runner has a `_validate_actions_fees` method that performs similar checks: it verifies whether the token may be used to pay fees, sums all fees while converting deposit-based tokens to their HTR equivalent through `NCFee.__get_htr_value__`, and compares the result to the expected value calculated from the count of fee-related Actions. The fees are then subtracted from the contract balance, likewise reflecting that they are burned.

We unify this behavior by removing both `FeeHeader.total_fee_amount` and `NCFee.__get_htr_value__` and introducing a new standalone function, `aggregate_fee_charges`. It checks whether each token may be used to pay fees and sums the charges using the appropriate conversions. It relies on the new fee policies model and is already prepared to receive the new token policies (**B8**, such as hUSDC): policy tokens charge the value described in their own policy, while deposit-based tokens charge the value derived from the HTR conversion. The only step left in either context is then to subtract the fees from the respective balance.

## Feature Activation

New policies should be introduced through Feature Activation (**B7**): we add a new version under the `FEE_POLICIES` setting and enable it in a new verification method. The policy version then propagates through the verification path for UTXO accounting, and through the Runner for Nano accounting. More details are given below.

## Deposit Addresses
[deposit-addresses]: #deposit-addresses

We must assert at configuration time that HTR policies do not define a `deposit_address` (**B6**), since HTR fees are always burned (**B2**).

For fee policies with a configured `deposit_address`, we must not subtract the fee values from the transaction's input/output balance. Instead, the fees should be present in the outputs, and a new verification method must ensure they are sent to the `deposit_address`. This method should assert that the exact fee value, in the correct token, is paid to the respective `deposit_address`, and it must skip outputs with token authorities, non-standard scripts, and timelocks.

On Nano, instead of merely subtracting the fee values from the contract's balance, we should also add them to the global balance. Since that feature is not yet implemented, we must decide between the following options:

1. Postpone this project until after the global balance is implemented; or
2. Always burn Nano fees until the global balance is implemented, regardless of the policy; or
3. Reject Nano fees paid in stablecoins, preserving the current behavior of accepting only HTR or deposit-based tokens, until the global balance is implemented.

## Nano Runtime

This section describes how the project affects the Nano Runtime.

### Nano Runtime Version

We should reuse the existing `NanoRuntimeVersion` system to introduce new policies, deriving a `FeePolicyVersion` from a `NanoRuntimeVersion` and using a single Feature Activation process to advance both (**B7**). For example, the protocol is currently on `FeePolicyVersion.V1` and `NanoRuntimeVersion.V2`. When we decide to introduce `FeePolicyVersion.V2`, we do so together with `NanoRuntimeVersion.V3`, deriving the `FeePolicyVersion` from the `NanoRuntimeVersion` inside the Runner.

### Nano Settings

The `NanoSettings` object returned from the `get_settings()` syscall currently has a single attribute, `fee_per_output`, mirroring the current settings model of the protocol. We should update it to use the new fee policies model. To do so, we must confirm that no Blueprints use this syscall on mainnet; otherwise, the change must be gated behind Feature Activation.

### New duplicate token rule

In the UTXO context, we enforce that all fee entries in a fee header use distinct tokens, but no analogous rule exists for Nano Action fees. We should add it for consistency, but we must first verify whether any mainnet transactions would violate the rule, in which case it should be enabled only through Feature Activation. To be decided.

# Drawbacks
[drawbacks]: #drawbacks

N/A

<!--Why should we *not* do this?-->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

N/A

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?
-->

# Prior art
[prior-art]: #prior-art

As noted in the initial project document, Liquid has a similar proposal. Although its architecture differs from ours, there are lessons to be drawn from it: https://github.com/ElementsProject/ELIPs/blob/main/elip-0204.mediawiki

<!--

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For protocol, network, algorithms and other changes that directly affect the
  code: Does this feature exist in other blockchains and what experience have
  their community had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other blockchains, provide readers of your RFC with a fuller
picture. If there is no prior art, that is fine - your ideas are interesting to
us whether they are brand new or if it is an adaptation from other blockchains.

Note that while precedent set by other blockchains is some motivation, it does
not on its own motivate an RFC. Please also take into consideration that Hathor
sometimes intentionally diverges from common blockchain features.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. How should fee accounting work on Nano while the global balance is not yet implemented? See the [Deposit Addresses](#deposit-addresses) section.
2. Do any Blueprints on mainnet call the `get_settings()` syscall, and must the resulting `NanoSettings` change therefore be gated behind Feature Activation? See the [Nano Settings](#nano-settings) section.
3. Would any mainnet transactions violate the duplicate token rule for Nano Action fees, and must it therefore be enabled through Feature Activation? See the [New duplicate token rule](#new-duplicate-token-rule) section.

<!--
- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
-->

# Future possibilities
[future-possibilities]: #future-possibilities

N/A

<!--
Think about what the natural extension and evolution of your proposal would be
and how it would affect the network and project as a whole in a holistic way.
Try to use this section as a tool to more fully consider all possible
interactions with the project and network in your proposal. Also consider how
this all fits into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
-->
