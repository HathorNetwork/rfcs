- Feature Name: Feature Activation for Transactions
- Start Date: 2023-08-09
- Initial Document: [Feature Activation](https://docs.google.com/document/d/1IiFTVW1wH6ztSP_MnObYIucinOYd-MsJThprmsCIdDE/edit)
- Author: Gabriel Levcovitz <<gabriel@hathor.network>>

# Summary
[summary]: #summary

This document describes a way to use the Feature Activation process with Transactions, complementing the existing implementation for Blocks. The original [Feature Activation for Blocks RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#evaluation-interval) can be read for more detailed information on what the Feature Activation process is, why it is necessary, and how it works.

# Motivation
[motivation]: #motivation

Implementing Feature Activation for Transactions was a requirement from the beginning, but during development of the initial RFC, it was determined that its complexity would be better addressed in a separate document. While the former defines a way to retrieve feature states for blocks, allowing for changes in block processing (block verification, block consensus, etc), the latter defines analogous behavior to retrieve feature states for transactions. This will be necessary for some of the main known use cases of the Feature Activation process, for example eventually releasing nano contracts.

# Guide-level explanation
[Guide-level explanation]: #guide-level-explanation

## Overview
[Overview]: #overview

The central idea to solve the calculation of feature states for transactions is to actually use the existing block process that handles all general requirements and is already implemented and tested.

To use block features states to retrieve feature states for transactions, let's first consider the possible state values. Blocks use multiple states to represent the Feature Activation process, such as `STARTED`, `MUST_SIGNAL`, `LOCKED_IN`, `ACTIVE`, etc, as they're responsible for the whole logic of the system. In the context of transactions, contrary to blocks, not all these states are relevant. In fact, for a transaction, it only matters if a feature is either `ACTIVE` or `INACTIVE`. For brevity, from this point forward when we say that a block is `ACTIVE`, we mean that its state for a certain feature is `ACTIVE`, and when we say that a block is `INACTIVE`, we mean that its state for a certain feature is any state _but_ `ACTIVE`.

Other than Feature Activation for Blocks, another existing system will be used as an inspiration for the Feature Activation for Transactions design, that is the Reward Lock system (called RL from now on). As will be described below, the way this system works can be mostly described as an analogy to the Feature Activation for Transactions (called FATX from now on), with some differences.

In loose terms, the problem RL solves is mostly determining the validity of some transaction in relation to the height of the **current best block**. This has to be calculated and determined in two different contexts: when a transaction is received and is in the mempool, and when a reorg happens. The FATX problem is essentially the same, but the best block's feature states are relevant, instead of its height.

Considering the reorg case, we can find a difference between the two systems, that is the FATX has one extra "dimension" when compared to the RL. For RL, a reorg is only relevant if it decreases the height of the best block (which could invalidate a previous valid transaction by re-locking the reward it tries to spend). For FATX, it doesn't matter if the best block's height is changed, it only matters if its state is changed, which could happen even if the best blockchain got larger after the reorg.

In other words, any time a reorg changes the state of the best block either from `INACTIVE` to `ACTIVE` or from `ACTIVE` to `INACTIVE`, some transactions may become invalid. Also, such change can occur if and only if a boundary block participates in the reorg. Otherwise, by definition in the Feature Activation for Blocks, the state will remain the same.

Another difference between RL and Feature Activation is their "runtime". Considering the lifecycle of a vertex in the full node, it is first received as a struct, or a byte array, and then it is parsed into one of the `BaseTransaction` subclasses, such as `Block` or `Transaction`. Only then its verification process starts, considering different rules for blocks and transactions, and RL validation is calculated.

In other words, the verification (or validation) of RL is only made _after_ the vertex bytes are parsed. For Feature Activation, that is not the case. Feature Activation must also be available _before_ bytes are parsed, as it must support changes such as updating `TxVersion`'s possible values, allowing for example for the release of nano contracts. Since those values are used to determine whether the vertex bytes are even parseable in the first place, Feature Activation must be available then. In that case, Feature Activation for Blocks must be used to determine the current state of the network for some feature before parsing the new vertex (using the current best block), and then FATX rules (described below) will be applied normally to verify the validity of the parsed `Transaction`. This will be further detailed.

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

In this section, technical details are expanded for what was described above.

## Reward Lock

Before detailing FATX, let's describe how the RL works in general terms. Then, we'll be able to observe the analogy and define FATX. As explained before, there are two main contexts for calculating RL (new vertices are separated into txs and blocks).

#### Dealing with new txs

1. A tx is received, and it spends the reward of a block. It is verified for reward lock. Then, there are two possibilities:
   1. If the current best block's height IS NOT enough to unlock the reward, the tx is invalid and is rejected.
   2. If the current best block's height IS enough to unlock the reward, the tx is valid and remains in the mempool.

#### Dealing with reorgs

1. A reorg happens. Then, if the new best height is lower than the best height before the reorg,
   1. Both txs that were already in the mempool, and confirmed txs that came back to the mempool, may have become invalid. For that, all txs in the mempool are re-verified (only for reward lock). If they're invalid, they're marked as such and removed from the mempool and the storage.

#### Dealing with new blocks

1. When a block is received, if one of the txs it confirms tries to spend a locked reward, the block is invalid and is rejected.

This rule is only for guaranteeing no rewards can be spent too early. In practice, it's impossible for such tx to be in the mempool, as it would have been invalidated before by the previous rules. TODO: Is this correct? Why does this rule exist, is it for compatibility with the previous reward lock mechanism?

## Feature Activation for Transactions

Now, before describing the contexts above for FATX, let's make some definitions.

### Premises

1. After an `ACTIVE` block, all next blocks will also be `ACTIVE` (considering the same blockchain).
2. Feature states for txs are only `ACTIVE` or `INACTIVE`.

### Requirements

Using the premises above, we must define a function that returns a state, given a transaction (note: we actually return multiple states for multiple features). That function must satisfy the following requirements:

1. All txs that are received after (time-wise) an `ACTIVE` block in the best chain, must also be `ACTIVE`.
2. When all txs are ordered by timestamp, there must not be an `INACTIVE` tx after an `ACTIVE` tx.

### Definitions

We then define the feature state for transactions function:

1. A tx is considered `ACTIVE` if
   1. It confirms a tx that is `ACTIVE`, OR
   2. It confirms a tx that has an `ACTIVE` first block.

Otherwise, it is `INACTIVE`. This guarantees that a tx will never be `INACTIVE` if there's an `ACTIVE` tx "before" it (to its left).

### Reward Lock analogy

Tooled with the definitions above, we're ready to extract the analogous contexts from RL to FATX, understanding how the FATX mechanism will work.

#### Dealing with new txs

1. A tx is received. It is verified for FATX, that is, the state function defined above is called, and there are two possibilities:
   1. If the current best block is `INACTIVE`, the tx is valid and remains in the mempool.
   2. If the current best block is `ACTIVE`, and
      1. If the tx is `INACTIVE`, it is invalid and is rejected.
      2. If the tx is `ACTIVE`, it is valid and remains in the mempool.

For RL, the tx is invalid only if the best block's height is not enough. For FATX, the tx is invalid only if the best block is `ACTIVE` and the tx is `INACTIVE` (note: this must be verified for all features).

#### Dealing with reorgs

1. A reorg happens. Then, if a boundary block participates in the reorg (that is, there was a boundary block in either the previous or the new best chain):
   1. Both txs that were already in the mempool, and confirmed txs that came back to the mempool, may have become invalid. All txs in the mempool must be invalidated and removed from the storage.

For RL, if the best chain decreases, all txs in the mempool are re-verified for reward lock. For FATX, if the best block's state changes, all txs in the mempool are invalidated and removed.

Why is the FATX case more extreme? For RL, a simple RL re-verification guarantees that the new best chain conforms to the tx validation requirements. However, as FATX may affect pre-parsing validation, it's possible that a tx becomes invalid even if its FATX verification does not find any errors. Let's look at an example.

When updating the `TxVersion` through the Feature Activation process, we can either add or remove values. For example, we could add a new value for a new nano contract, but we could also eventually remove it. This creates a symmetry, having the same consequence for both situations:

- We set up the removal of an existing NC, and then the best block changes to `ACTIVE`, removing support for that NC
- We set up the addition of a new NC, and then the best block changes back to `INACTIVE`, removing support for that NC

Therefore, in both cases we may end up with txs in the mempool that were parsed with the now removed NC's `TxVersion`.

If we were to simply re-run FATX verification for all txs in the mempool, like it's done for RL, the tx could still be considered valid for its state, but in fact it shouldn't even exist, as it could not have been parsed considering the new current `TxVersion` possible values. We could store the tx's raw bytes, and then reparse and re-verify all of it, but for simplicity we just invalidate and remove all txs from the mempool, so the sync algorithm naturally re-syncs valid txs considering the updated `TxVersion`.

There is indeed a performance penalty for throwing away the whole mempool, but the relevant fact here is that this can only happen when the reorg affects a boundary block, which exists only every two weeks. Therefore, it's guaranteed that this mempool throwaway will not happen during the two-week evaluation interval. Even at the boundaries, it will only happen if by chance the boundary block is part of a reorg, AND if the best block state transitions from or to `ACTIVE`. Therefore, it's guaranteed that the mempool throwaway and re-sync will happen **less than every two weeks**, which seems like a reasonable tradeoff. In fact, it will happen much less, as we'll likely only have a new feature transitioning to (or from) `ACTIVE` every few months.

#### Dealing with new blocks

1. An `ACTIVE` block can only confirm txs that are also `ACTIVE` (except for the first `ACTIVE` block).

This guarantees that no `INACTIVE` txs are accepted after the first `ACTIVE` block is received.

## Retrieving Feature States for Transactions

To support the mechanism described above, a new `feature_states` metadata attribute will be introduced for transactions. It will be calculated according to the rules above and set in `BaseTransaction.update_initial_metadata()`. To calculate the feature state for a new tx, we first get this metadata from its parents. If any of them is `ACTIVE`, we return `ACTIVE`. Otherwise, we get the states of its parents' first block. If any of them is `ACTIVE`, we return `ACTIVE`. Otherwise, we return `INACTIVE`. The same function is also used for verification.

### Mutability of the `feature_states` metadata

There are two situations that could result in the need of updating this metadata. Instead, we want a solution where this metadata is immutable, for simplicity.

#### When a reorg happens

In this case, since affected transactions are discarded from the mempool, there's no need to update the metadata. The txs will be re-synced and their metadata will be calculated accordingly, as if they were new txs.

#### When one of our parents is confirmed by an `ACTIVE` block

When a parent tx is confirmed by an `ACTIVE` block, it's possible that our metadata would have to be updated. There are two sub-cases:

1. This block is NOT the first `ACTIVE` block in the network
   1. In this case, we would have to be `ACTIVE` in the first place, according to the FATX validation rules. Therefore, no metadata update is necessary.
2. This block IS the first `ACTIVE` block in the network
   1. In this case our metadata could transition from `INACTIVE` to `ACTIVE`. If we did this, we would have to update the metadata and re-verify all our children. Instead, we'll mimic the reorg case, and force a purge and re-sync of the mempool. This is also very rare, as it only happens **once** for each feature.

Other alternatives involve keeping track of mutable metadata states which introduces complexity. Also, it would have been necessary to update `HathorManager.get_new_tx_parents()` according to the FATX rules, so `ACTIVE` txs are returned when necessary. Otherwise, we could end up with `INACTIVE` txs in the mempool that are impossible to confirm.

# Drawbacks
[drawbacks]: #drawbacks

The drawback to this solution is the necessity of purging the mempool in some specific cases. As explained above, these cases are very rare, so this purging is guaranteed to not happen more often than once every two weeks. Actually, it's very likely it will only happen once every few months, if it ever happens.

# Rationale and alternatives
[Rationale and alternatives]: #rationale-and-alternatives

This RFC passed through a lot of completely different idea iterations before arriving at this solution.

The first idea that was discarded right from the beginning was using signal bits for transactions, instead of leveraging the Feature Activation for Blocks. This would mean a completely separate process for transactions, and a lot of the already implemented requirements would have to be reimplemented. It made much more sense to leverage the existing block mechanism.

Then, a whole other group of ideas was studied, that was trying to find a simple way of choosing an "associated feature block" for each transaction. The feature state of a transaction would simply be the feature state of that associated block. The problem was finding an associated block that makes sense, and that works when reorgs happens, while satisfying the requirement that the state of transactions cannot "retreat" from `ACTIVE` to `INACTIVE` when transactions are ordered by timestamp. We tried a lot of different options, using best blocks, first blocks, first blocks of parents, introducing a delay of an evaluation interval between a tx and its associated block, etc. After a lot of work analysing those options, we would always find a case where the requirements would be broken and the solution became convoluted. Finally, taking inspiration from the existing Reward Lock mechanism seemed like the right path.

# Prior art
[prior-art]: #prior-art

I couldn't find any prior art relevant to this RFC, considering Hathor's unique DAG architecture for transactions. While the Feature Activation for blocks is heavily based on the Bitcoin solution, the transactions solution is very specific to Hathor.

# Task breakdown

Here's a table of main tasks:

| Task                                     | Dev days |
|------------------------------------------|----------|
| Create the new `feature_states` metadata | 1        |
| Implement verification of new txs        | 2        |
| Implement dealing with reorgs            | 2        |
| Implement dealing with new blocks        | 1        |
| **Total**                                | **6**    |
