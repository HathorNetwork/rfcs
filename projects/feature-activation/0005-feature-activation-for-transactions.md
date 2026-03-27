- Feature Name: Feature Activation for Transactions
- Start Date: 2023-01-23
- Initial Document: [Feature Activation](https://docs.google.com/document/d/1IiFTVW1wH6ztSP_MnObYIucinOYd-MsJThprmsCIdDE/edit)
- Author: Gabriel Levcovitz <<gabriel@hathor.network>>

# Summary
[summary]: #summary

This document describes a way to use the Feature Activation process with Transactions, complementing the existing implementation for Blocks. The original [Feature Activation for Blocks RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md) can be read for more detailed information on what the Feature Activation process is, why it is necessary, and how it works.

# Motivation
[motivation]: #motivation

Implementing Feature Activation for Transactions was a requirement from the beginning, but during development of the initial RFC, it was determined that its complexity would be better addressed in a separate document. While the former defines a way to retrieve feature states for blocks, allowing for changes in block processing (block verification, block consensus, etc), the latter defines analogous behavior to retrieve feature states for transactions. This will be necessary for some of the main known use cases of the Feature Activation process, for example eventually releasing nano contracts.

# Guide-level explanation
[Guide-level explanation]: #guide-level-explanation

## Overview
[Overview]: #overview

The central idea to solve the calculation of feature states for transactions is to actually use the existing block process that handles all general requirements and is already implemented and tested. By doing that, we can define feature states for transactions as simply a "forward" from the feature states of some block.

This problem demonstrated to be harder than believed in the beginning of this work, resulting in multiple completely different ideas being explored. Some of those became RFCs by themselves, now kept just for reference in an [iterations directory](./0005-iterations). Eventually we would find problems in each solution, and only by leveraging those initial iterations we arrived at the proposal in this document. The other ideas were convoluted and complex, generating multiple edge cases that were hard to keep track and error-prone. Considering that, some of the text was extracted from the previous documents.

To use block features states to retrieve feature states for transactions, let's first consider the possible state values. Blocks use multiple states to represent the Feature Activation process, such as `STARTED`, `MUST_SIGNAL`, `LOCKED_IN`, `ACTIVE`, etc, as they're responsible for the whole logic of the system. In the context of transactions, contrary to blocks, not all these states are relevant. In fact, for a transaction, it only matters if a feature is either `ACTIVE` or `INACTIVE`. For brevity, from this point forward when we say that a block is `ACTIVE`, we mean that its state for a certain feature is `ACTIVE`, and when we say that a block is `INACTIVE`, we mean that its state for a certain feature is any state _but_ `ACTIVE`.

After exploring the previous ideas, I determined that a _theoretical_ great solution would be to add a block parent to all transactions. The state of a transactions would be the state of its parent block. That would mean we wouldn't have to deal with reorgs, which is the main source of difficulties, since voiding a parent block would automatically void its transaction dependencies. However, in practice, making this change would be too risky as it could affect multiple dynamics of the network.

Then, we arrived at the solution proposed in this document, which has most of the same advantages from the parent block solution, however it does not require any breaking change whatsoever. Here it is: **for a feature, a transaction is considered `ACTIVE` if its closest block, that is, its ancestor block with the greatest height, is `ACTIVE`**. This will be detailed in the Reference-level section.

In other words, transactions today can already point to blocks via their inputs. Only a small portion of transactions spend from blocks, though. The idea is then to propagate this indirect block dependency to all transactions. This can be achieved by simply creating a new metadata that is set when a transaction is received from the network. The calculation is deterministic, equal for all peers, and with constant execution complexity. Also, reorgs are handled automatically by existing code, and no change has to be made in critical components such as the consensus.

Equipped with this definition, the solution is complete. No other edge cases have to be handled. Also, to be clear, no change in the Feature Activation for Blocks is necessary. There are some use case limitations, though, explained in the [drawbacks] section.

## Example - Releasing new Nano Contracts

To illustrate the usage of Feature Activation for Transactions, we'll demonstrate how it would be used to release a new Nano Contract.

To release a new Nano Contract, a new value would be added to the `TxVersion` enum, allowing deserialization of a different kind of transaction. Since the `FeatureService.is_feature_active_for_transaction()` method requires a `Transaction` instance, it cannot be accessed during deserialization, as the instance hasn't been construct yet. The deserialization will always succeed, even if the Feature fails to become `ACTIVE`.

Then, a new method must be added in the verification phase of `BaseTransaction`, to assert that the deserialized `TxVersion` is valid. In that method, `is_feature_active_for_transaction()` would be called, and depending on the state returned, the `TxVersion` representing the new Nano Contract would be accepted or not.

This demonstrates that there are two possible known use cases for Feature Activation for Transactions:

1. Changing behavior in transaction validation. In this case, the usage is straight forward, simply a call to `FeatureService.is_feature_active_for_transaction()` to check whether the new feature is `ACTIVE`.
2. Changing behavior in transaction deserialization. In this case, it may be necessary to create a new transaction validation method that verifies the validity of using a new deserialization feature.

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

In this section, technical details are expanded for what was described above. Before detailing the solution, let's formalize the context.

## Overview

### Premises

1. After an `ACTIVE` block, all next blocks will also be `ACTIVE` (considering the same blockchain).
2. Feature states for transactions are only `ACTIVE` or `INACTIVE`.

### Requirements

Using the premises above, we must define a function that returns a state, given a transaction (note: we actually return multiple states for multiple features). That function must satisfy the following requirements:

1. A while after one `ACTIVE` block in the best chain, it must be possible to create `ACTIVE` transactions.
2. A transaction that has an `ACTIVE` transaction as a dependency, is also `ACTIVE`.

### Definition

Repeated here from the Guide-level section:

**For a feature, a transaction is considered `ACTIVE` if its closest block, that is, its ancestor block with the greatest height, is `ACTIVE`.**

## Retrieving Feature States for Transactions

Analogously to the Feature Activation for Blocks, the state for a transaction will be retrieved from a method in the `FeatureService`. Considering the definition above, a reference-implementation is provided:

```python
class FeatureService:
    def is_feature_active_for_transaction(self, *, tx: 'Transaction', feature: Feature) -> bool:
        """Return whether a Feature is active for a certain Transaction."""
        metadata = tx.get_metadata()
        closest_ancestor_block = self._tx_storage.get_block(not_none(metadata.closest_ancestor_block))

        return self.is_feature_active_for_block(block=closest_ancestor_block, feature=feature)
```

## Handling reorgs

Since the transaction's state is calculated based on an existing direct or indirect block dependency of the transaction, it's guaranteed that when this block is voided in a reorg, the transaction will also be voided via current consensus rules. Therefore, no specific handling of reorgs is necessary.

It's also interesting to mention that since there's a requirement of 300 blocks between a transaction and the block it spends (via the Reward Lock mechanism), it's extremely unlikely that transactions are voided for this reason.

## New `closest_ancestor_block` metadata

A new metadata field must be created to propagate the `closest_ancestor_block` information forward in the DAG. Every time a transaction is received, its `closest_ancestor_block` metadata must be calculated from its dependencies (parents and inputs). If the dependency is a block, the block itself is a `closest_ancestor_block` candidate. If the dependency is a transaction, its own `closest_ancestor_block` is a candidate. Given all candidates, the block with the greatest height is defined as the `closest_ancestor_block`. This calculation is O(1) as the number of dependencies is bounded. It's guaranteed that this will be the same for all peers, as dependencies are part of the transaction's core structure and are validated. It's also required that the full node has all dependencies downloaded before accepting a new transaction.

Here's a reference implementation for calculating `closest_ancestor_block` metadata, defined in the `BaseTransaction` class:

```python
  def _calculate_closest_ancestor_block(
      tx: 'Transaction',
      settings: HathorSettings,
      vertex_getter: Callable[[VertexId], 'BaseTransaction'],
  ) -> VertexId:
      """
      Calculate the tx's closest_ancestor_block. It's calculated by propagating the metadata forward in the DAG.
      """
      from hathor.transaction import Block, Transaction
      if tx.is_genesis:
          return settings.GENESIS_BLOCK_HASH

      closest_ancestor_block: Block | None = None

      for vertex_id in tx.get_all_dependencies():
          vertex = vertex_getter(vertex_id)
          candidate_block: Block

          if isinstance(vertex, Block):
              candidate_block = vertex
          elif isinstance(vertex, Transaction):
              vertex_candidate = vertex_getter(vertex.static_metadata.closest_ancestor_block)
              assert isinstance(vertex_candidate, Block)
              candidate_block = vertex_candidate
          else:
              raise NotImplementedError

          if (
              not closest_ancestor_block
              or candidate_block.static_metadata.height > closest_ancestor_block.static_metadata.height
          ):
              closest_ancestor_block = candidate_block

      assert closest_ancestor_block is not None
      return closest_ancestor_block.hash
```

For genesis transactions, by definition, the genesis block is their `closest_ancestor_block`.

A migration will also be necessary to populate this new metadata for existing transactions.

## Mutability of transaction parents

One complication factor that we haven't considered yet is the fact that the parents of a transaction can be tempered with, as they're not part of the transaction signature. This means that after a transaction is pushed to the network, a third-party can re-push a copy of that transaction, only changing its parents and weight, for example. Then, a third party could manipulate a transaction in such a way that the transaction's feature state is toggled (to/from `ACTIVE` from/to `INACTIVE`) if the third-party transaction has a higher weight than the original transaction.

Let's consider an example. A feature is created to activate a new Nano Contract, that is, a new `TxVersion` will become allowed during deserialization. A transaction using the new Nano contract is pushed to the network, point to parent that have this feature `ACTIVE`, and is accepted by the network. Then, a third-party copies this transaction, increasing its weight and changing its parents to `INACTIVE` transactions, and pushes it to the network. That copy would win over the original transaction, voiding it (for its weight), but since from the perspective of the copy the new Nano Contract is not activated yet, that transaction would not even be accepted in the first place. Therefore, the original transaction would remain valid and accepted. An analogous example can be created for the opposite situation (an original `INACTIVE` transaction being shifted by a third-party to become `ACTIVE`).

In other words, if a third-party tries to manipulate a transaction such that its activation is toggled, the copied transaction will only remain valid if the usage of a feature doesn't affect the validity of that transaction in the first place.

Therefore, the mutability of transaction parents is not an issue for the solution proposed in this document.

## Selection of transaction parents

Currently, wallets do not select transaction parents by themselves. This is done by the `tx-mining-service`, through a full node API, that chooses parent transactions randomly from the tips of the DAG. An open question in this RFC is whether we should change this behavior, and somehow prioritize parent transactions with `ACTIVE` states. This would help make sure new transactions can use the newest features, when they're available. It could also be possible to request a specific feature to be active and select the parents accordingly.

# Drawbacks
[drawbacks]: #drawbacks

### Reliance on miners

A drawback is that for a feature to be available for transactions, it's necessary that a block reward is spent. Only then the metadata will propagate for all new transactions in the network. This means that it would be theoretically possible for all miners to collude and decide to hold their block UTXOs, rendering the transaction feature in an "on hold" state until a block reward is spent.

This is extremely unlikely however, as there's a natural incentive for miners to spend their rewards, especially considering most of our partners are mining pools that have to share rewards with their clients. It also wouldn't make sense for miners to collude in such a way, as they could simply send negative bit signals, preventing the feature activation in the first place. If miners vote for the activation of a feature, it's expected that they would be ready to support it and would continue with their normal operations, which include spending block rewards. This usually happens in the same day a block is found.

### Relaying transactions from the past

Currently, it is possible to relay new transactions spending and/or confirming old transactions. This means that even after a feature becomes active, enabling new rules, it's possible to create a new transaction that is inactive, that is, still uses the old rules. The restriction is that the new transaction must only spend and confirm old transactions, so they will also be inactive. Therefore, the Feature Activation requirements are respected (the new inactive transaction will not depend on an active transaction).

This means that it's not possible to enforce a new feature on old transactions using this mechanism. For example, it would not be possible to remove the validity of an existing opcode, as someone could simply create a transaction spending and confirming transactions from before this removal was activated. For the known use cases, this is not an issue. Every planned feature at this moment can be activated through the solution in this document. A solution for removing existing features will be defined in a separate RFC, in the future, if necessary.

# Rationale and alternatives
[Rationale and alternatives]: #rationale-and-alternatives

This RFC passed through a lot of completely different idea iterations before arriving at this solution. Please see the [iterations directory](./0005-iterations) for more information.

# Prior art
[prior-art]: #prior-art

I couldn't find any prior art relevant to this RFC, considering Hathor's unique DAG architecture for transactions. While the Feature Activation for blocks is heavily based on the Bitcoin solution, the transactions solution is very specific to Hathor.

# Future possibilities

- As explained in the "Relaying transactions from the past" drawback section, a solution for removing existing features will have to be defined in a separate RFC, in the future, if necessary.

# Task breakdown

Here's a table of main tasks:

| Task                                       | Dev days |
|--------------------------------------------|----------|
| Implement state retrieval for transactions | 0.5      |
| Implement tests and simulations            | 1        |
| **Total**                                  | **1.5**  |
