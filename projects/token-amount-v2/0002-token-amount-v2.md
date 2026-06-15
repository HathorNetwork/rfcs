- Feature Name: token-amount-v2
- Start Date: 2026-06-02
- Initial Document: [Add More Decimals to Tokens](https://docs.google.com/document/d/1D7eq6DHsHTsisCUF2mQc8rW2MWs60EODL8TyzdHMUKU/edit?tab=t.0)
- Author: Gabriel Levcovitz <gabriel@hathor.network>

# Summary
[summary]: #summary

This project aims to upgrade the protocol to support greater decimal precision for tokens (more than the current two decimals).

# Motivation
[motivation]: #motivation

Currently, tokens on Hathor only support two decimal places. This creates challenges for assets originating on networks with greater precision. For example, porting BTC to Hathor means the smallest transferable value is 0.01 BTC, which is impractically large for micropayments, trading, and interoperability use cases. Most token standards rely on 6, 8, or even 18 decimals. Expanding decimal support will make Hathor more interoperable, competitive, and appealing for developers and projects porting tokens.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Requirements

We begin by enumerating a few business decisions stemming from the initial project document, linked above:

- **B1 (Business decision 1):** Token protocol is upgraded to support more than two decimal places (targeting compatibility with standards like ERC-20’s 18 decimals).
- **B2:** Token creators cannot choose the number of decimals. It should be the same for all tokens in the network, including HTR and all custom tokens.
- **B3:** Wallets, explorer, and APIs correctly display and process the new decimal precision.
- **B4:** The migration and upgrade process ensures existing tokens and balances are safely transitioned or clearly handled.
- **B5:** All existing tokens should be "upgraded" to also support this number of decimals, including HTR.

There was also discussion over how many decimal places we would adopt. From the approved [analysis document](https://github.com/HathorNetwork/rfcs/pull/113), we settled on:

- **B6**: Use 18 decimal places encoded in 1-byte length-prefix varints.

For clarity and simplicity, this project is referred to as Token Amounts V2, where the current state is V1 with 2 decimal places and the proposal is V2 with 18 decimal places.

## Upgrade Surface

This is a foundational upgrade that touches most areas of the full node codebase — and also most components in our ecosystem, such as wallets and the explorer, although those are out of scope for this RFC. The following is an enumeration of the protocol-level and node-level items that must be upgraded in this project:

### Wire encoding

The current encoding of token amounts for the wire uses a custom varint of 4 or 8 bytes, with support for values of up to `2^63`, which includes the 2 decimal places. A new varint codec must be added to support the new range of amounts.

- **I1 (Implementation decision 1):** Use a 1-byte length-prefix varint (from the approved [analysis document](https://github.com/HathorNetwork/rfcs/pull/113)).
- **I2:** The maximum amount of tokens per output will remain exactly the same but normalized to 18 decimals, that is, `2^63 * 10^16`, where `10^16` is hereby named the **normalization factor** — the factor used to scale V1 decimals to V2 decimals (from 2 to 18 decimal places).

### Vertex versioning

Existing vertices on the network form an immutable signed history with V1 token amounts that cannot be changed or migrated. We must add a versioning scheme that supports both V1 and V2 vertices concurrently, while still allowing them to interact with each other.

- **I3:** Blocks do not need 18 decimal places for their rewards, so they will remain V1-only. This avoids the need to upgrade block templates and mining rigs.
- **I4:** Transactions can be either V1 or V2, with the version indicated by a signal bit on the transaction (detailed in the [Reference-level section](#i4---versioning-transactions)). Support for V2 transactions will be gated by a Feature Activation process.
- **I5:** A transaction's version dictates the version of all its token amount fields: output values, fee values (from the Fee header), and Nano action values (from the Nano header).

### Balance verification

Balance verification asserts that the sum of inputs equals the sum of outputs in a single transaction, considering the interactions between minting and melting tokens, fees, and Nano actions.

- **I6:** Both V1 and V2 transactions may spend V1 or V2 outputs, provided the balance is correct across all 18 decimal places. This means V2 values must never be truncated to V1 during accounting.
- **I7:** All internal full node arithmetic will be done in V2 with 18 decimal places, and V1 values will be scaled by the normalization factor. This allows the full node codebase to be upgraded to use V2 internally independently of support for V2 transactions, which is gated by I4.

### Custom tokens

For fee-based tokens, the fee header will support V2 values (already covered in I5), and no further discussion is required. For deposit-based tokens, we must determine how the deposit and withdrawal of HTR will work. Currently, 1% of the minted value is required in HTR (rounded up) and the same 1% can be withdrawn when melting (rounded down). For example, minting 1.00 TOK requires 0.01 HTR, and minting 1.01 TOK requires 0.02 HTR. Melting 2.00 TOK yields 0.02 HTR, and melting 1.99 TOK yields 0.01 HTR. This means deposits and withdrawals always change in steps of 0.01 HTR, and therefore we decide:

- **I8:** We keep the same step of 0.01 HTR for minting and melting custom deposit-based tokens, regardless of the increased precision. This keeps the economic model unchanged. For example, minting 1.001 TOK will require 0.02 HTR, and melting 1.999 TOK will yield 0.01 HTR. In other words, we apply the same 1% calculation, then round to the nearest HTR cent (up for deposits, down for withdrawals).

### Storage and indexes

Storage of versioned transactions requires no upgrade, as they will naturally use their encoded bytes as the stored artifact. Token-related indexes need to be upgraded to support the new token amounts. This includes the UTXO index and the Tokens index.

- **I9:** Indexes will use V2 values internally, as in I7. V1 values will be normalized accordingly.
- **I10:** The RocksDB implementation of the UTXO index currently uses a fixed slot of 8 bytes for storing amounts. It will be changed to use a varint slot with the same serialization as the V2 token amounts.

### APIs

The full node APIs must reflect the new versioned tokens. For example, the `/v1a/transaction` route must return output values as they are defined in the respective transaction, so:

- **I11:** We'll introduce new `/v2` API routes for all existing `/v1a` routes that receive amounts as requests or send them as responses.
- **I12:** The new `/v2` APIs will return amounts as strings, and include fields with the token amount version and number of decimal places.
- **I13:** Existing `/v1a` APIs will be kept but deprecated, and they will return errors for V2 transactions.

Clients can therefore update their API calls incrementally while the full node is already using V2 accounting internally, at least until V2 transactions are activated (per I4). At that point, clients must be fully updated.

### Nano contracts runtime

Nano contract actions are covered by I5: the transaction's version dictates the precision of action values. However, we must consider how differently-versioned amounts will affect the Nano runtime. The complicating factor for Nano is existing Blueprints. For example, an existing Blueprint whose code was written with hardcoded support for 2 decimal places cannot receive actions with 18 decimal places. This complication appears at multiple points of interaction:

1. Actions/fees coming from transactions into the Blueprint.
2. Actions/fees coming from the Blueprint into other contracts.
3. Retrieval of contract balances.
4. Retrieval of output values from the `VertexData`.
5. Nano method call arguments (which may represent token amounts — even if they use the `int` type, not just `Amount`), coming from both transactions and other contracts.

Therefore, we can use normalized values internally in all Runner accounting, contract balances, etc. (I7). However, values must be de-normalized when fed into a Blueprint and re-normalized when retrieved from it. The only way to perform this normalization safely is to prevent Blueprints developed for 2 decimal places from interacting with Blueprints developed for 18 decimal places.

- **I14:** Blueprints will also be token-amount-versioned. Following from I5, an OCB transaction with V1 creates a V1 Blueprint and an OCB transaction with V2 creates a V2 Blueprint.
- **I15:** V1 Blueprints can only be called by V1 transactions and other V1 Blueprints, and V2 Blueprints can only be called by V2 transactions and other V2 Blueprints. This ensures that all values in a single call chain are interpreted consistently, resulting in two separate Nano "worlds" that cannot interact directly. Runtime verification will be added to prevent cross-version calls (public, view, proxy) and balance retrieval.
- **I16:** Upgrading a contract's Blueprint should work normally, and can be used to upgrade a contract from a V1 Blueprint to a V2 Blueprint, enabling support for the new decimal precision. Since the balance is normalized internally, this requires only correctly normalizing (or not) the values at the boundaries, according to the Blueprint's version (and gated by I15).
- **I17:** We cannot drop support for V1 transactions while there are still V1 contracts in the network.
- **I18:** The serialization of values and amounts in the Nano state trie must be updated to support V2 values. This will break the current state root ID.

### Shielded outputs
[shielded-outputs]: #shielded-outputs

We must determine how the shielded outputs project will interact with this project.

Q: Can we put values with increased precision in shielded outputs? Does that increase the required size of proofs?
A: It does increase the Borromean range proof since its size grows linearly.

Q: Should we support shielded outputs with V1 values, or just V2? If we support both, how does normalization affect the cryptographic balance accounting? A: We must ship Token Amount V2 before shipping shielded outputs and enforce all shielded outputs to use V2. Otherwise, we can't balance inputs and outputs with different versions.


### Migration

- **I19:** We must require a sync from scratch for this upgrade, since there are internal backwards-incompatible changes in the storage (I10 and I18).

<!--
Explain the proposal as if it was already included in the network and you were
teaching it to another Hathor programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Hathor programmers should *think* about the feature, and how it
  should impact the way they use Hathor. It should explain the impact as
  concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or
  migration guidance.
- If applicable, describe the differences between teaching this to existing
  Hathor programmers and new Hathor programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section
should focus on how compiler contributors should think about the change, and
give examples of its concrete impact. For policy RFCs, this section should
provide an example-driven introduction to the policy, and explain its impact in
concrete terms.
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section details the low-level implementation decisions, and is still a work in progress. For now, only the details for I4 are included, as it is a design discussion point:

### I4 - Versioning Transactions
[i4-reference]: #i4---versioning-transactions

The requirement of concurrently supporting both V1 and V2 transactions on the network necessitates versioned transactions, since we cannot simply assume all transactions after a feature activation are V2, for example. We also require that existing transactions are interpreted as V1. There are a few options for versioning:

1. Set a signal bit on the transaction's first byte (currently unused), `0` for V1 and `1` for V2.
2. Create a new flag-only header: when absent, the transaction is V1; when present, it's V2.
3. Create new transaction versions (`TxVersion`).

In reverse order, option #3 is non-canonical and should be discarded, as it is not composable with other versioned features that may be introduced in the future. Currently, we have `TxVersion` 1 for regular transactions, 2 for token creation transactions, and 6 for OCBs. We would need three new versions, one for each transaction subtype, to represent each of them with token amount V2. Then, suppose a new versioned feature were added in the future, orthogonal to the token amount. We would need to create new `TxVersion`s for each combination of features. This does not scale.

Option #2 is viable, but unnecessarily wasteful. Adding a new flag-only header would require 1 byte for the header version (with an empty body). All V2 transactions would have to carry that extra 1 byte, on top of the already increased encoded token amount sizes.

Therefore, option #1 is the proposed approach: repurpose a signal bit in the transaction's first byte to encode the token amount version. The first byte of all vertices is a reserved signal byte intended precisely for this kind of upgrade. On mainnet/testnet, the least significant nibble of the signal byte of blocks is used for voting on Feature Activation processes. For transactions, the entire signal byte remains unused.

We use the least significant bit of a transaction's signal byte, with `0` representing V1 and `1` representing V2. If more token amount versions are ever needed, additional bits can be repurposed. This makes the token amount versioning effectively cost-free.

The only drawback is that empty signal bits are not currently enforced on transactions. We would first need to add a new verification that prohibits transactions with unknown signal bits, to prevent old transactions with a set bit from being interpreted as V2 in the future. A scan was performed on both mainnet and testnet, and no existing transactions violate this rule.

Lastly, Feature Activation would be used to add support for V2 transactions: the project can be fully implemented with the feature disabled — V2 transactions are rejected at verification. This means there would be no V2 output values, fees, Nano actions, or OCBs. Meanwhile, the internal changes can be made and clients can upgrade. When everything is ready, the feature is activated and support takes effect system-wide.

<!--
This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.
-->

# Drawbacks
[drawbacks]: #drawbacks

1. A slight increase in storage and bandwidth from the larger encoded token amount sizes; however, the [analysis document](https://github.com/HathorNetwork/rfcs/pull/113) determined this to be negligible.
2. Requires a sync from scratch, including breaking the Nano state root ID.

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

N/A

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

N/A

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
