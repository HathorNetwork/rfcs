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

The central idea to solve the calculation of feature states for transactions is to actually use the existing block process that handles all general requirements and is already implemented and tested. By doing that, we can define feature states for transactions as simply a "forward" from the feature states of some block. Then, our problem becomes, which block to use for each transaction?

To understand this question, let's think about the block process. In general terms, the feature state for a block depends on the previous blocks, or the blocks "behind" it (meaning, in the past). For blocks, height and timestamp can be used interchangeably, as every block has a greater height and a greater timestamp when compared to its parent.

It's also a fact that while the block count advances, a feature state cannot "retreat". For example, the feature cannot be `ACTIVE` on one block and then go back to `STARTED` on the next block, as that state transition would be backwards. This is only true when looking for blocks in the best chain (or more generally, in any single blockchain). If instead we look for the best block over time, feature states can indeed retreat, when reorgs happen. After the reorg is complete, the "always-increasing" property of feature states is restored, as blocks with "retreating" states would be part of the voided blockchain.

For transactions, we have the same requirement. For every transaction that arrives, the feature state for that transaction cannot "retreat" from the feature state of any previous transaction (meaning, any transaction in the past). The complication here is that while blocks follow a linear pattern, transactions do not, as they're organized in a DAG (in other words, transactions don't have a height property). Therefore, contrary to blocks, transactions also have the requirement that feature states cannot retreat when looking for transactions over time. This means that at any point in time, any transaction must have a feature state that is the equal to or "greater" than the feature state of any transaction that arrived before it did.

This poses a complication for dealing with reorgs. When they happen, blocks are voided, but transactions are not — they may only return to the mempool. This would leave some transactions with features states retrieved from voided blocks. In other words, the transactions would hold the "retreated" state. To solve this problem, one possible solution would be to recalculate the feature states of transactions when a reorg happens, that is, we would have to choose another block for the transaction to retrieve its feature states from, because they could be invalidated after the reorg thanks to their own feature state. This would introduce unnecessary complexity, but there's another option.

Going back to the question in the beginning of this section, we must define some block that will be associated with some transaction, and to retrieve the feature states for that transaction, we'll simply retrieve the feature states for the associated block. It follows that for any new transaction, its associated block's height must be greater than or equal to the associated block's height of any past transaction. Considering what was described in the previous paragraph, an alternative solution to the reorg problem is adding another requirement for those associated blocks: their state must not change when a reorg occurs. If that is true, the transaction's feature states will always remain valid, even after a reorg.

Then, finding this associated block for each transaction is our main problem to be solved, and it guarantees that feature states will never "retreat".

There are a few obvious choices for choosing an associated block, like using the best block, the first block, or other related ideas, but they all introduce problems that will be explored in the following sections. The solution we'll arrive at leverages boundary blocks of Feature Activation evaluation intervals.

**The associated block for a transaction will always be the boundary block of the previous evaluation interval**. This introduces a delay of at least two weeks between a transaction and the block used to determine its feature states. Intuitively, that delay guarantees the requirement of "feature states of associated blocks cannot change", as the only thing that could change that block's feature states is a two-week reorg, and the probability of that happening is virtually zero.

---
THIS IS WRONG

Our question now becomes, how to retrieve that boundary block? First, we need to determine the boundary block that is closest to our transaction, to the left (in the past). Knowing block heights is an easy way to navigate through boundary blocks, but transactions do not have heights, only timestamps. So we need a way to bridge from the time domain to the height domain. Using the transaction's timestamp, we'll get the best block at that time. Now, with a block in our hands, we can get its height to easily convert between domains. Finally, to get to the boundary block of the previous evaluation interval, only simple height math is necessary.

---

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

In this section, technical details are expanded for what was described above.

## Rationale

### Definitions

1. We use the term tx to refer to a `Transaction`, not a vertex.
2. Considering the Feature Activation context, as [defined in the original RFC](https://github.com/HathorNetwork/rfcs/blob/master/projects/feature-activation/0001-feature-activation-for-blocks.md#evaluation-interval),
	1. An Evaluation Interval is a repeating period in the Feature Activation process. It's defined as `40320` blocks, the equivalent of 2 weeks given the average time between blocks.
	2. A Boundary Block is a block that starts an Evaluation Interval, that is, its height is a multiple of `40320`.
	3. Similarly, a Boundary Height is the height of a Boundary Block.
3. Both blocks and transactions can have different states for different features, but for simplicity sometimes we mention "the block's feature state" or "the transaction's feature state", in singular.

### Premises

1. We already have feature states for all blocks, which are readily available through the `FeatureService`.
2. We'll leverage that to define feature states for transactions. That is, the feature state for some tx is the same state as some block, so we need some function `f(tx) -> block`.
3. Analogously to blocks, whenever a new tx arrives, its feature state must not "retreat" when compared to the feature state of a tx that arrived before it did. In other words, for a `tx2` that arrives after a `tx1`, that is, `tx2.timestamp >= tx1.timestamp`, we require that `f(tx2).feature_state >= f(tx1).feature_state`. This must hold true even after reorgs.

Note: in the context of feature states, `>=` means that a state can only occur after another state in the state machine, or it is the same state.

### The Problem

Given the definitions and premises above, our problem becomes a matter of finding a function `f(tx)`. We'll iterate over possible solutions, identifying issues and fixing them until we reach a working result.

The first obvious solution would be for `f(tx)` to simply return the best block, using `TransactionStorage.get_best_block()`. There are also obvious issues with this option: as the best block changes over time, the feature state of a tx would become transient.

An option that solves this issue is using `TransactionStorage.get_best_block_tips()` instead. This method receives a `timestamp`, that would be the transaction's timestamp. In other words, we retrieve the best block at the time of the transaction, removing the transient component. Let's call this "best block at the time of the transaction" the "transaction's best block", for brevity.

The issue in this case would be dealing with reorgs. Let's consider the following timeline:

```
   b1   tx1
---|-----|---> time
```

We start with `b1` as the best block, and then `tx1` arrives. During the verification process of `tx1`, some feature state is queried. Using the solution described above, `tx1`'s feature state would be the same as `b1`'s feature state, let's say it is `ACTIVE`.

Then, there's a reorg and `b1` is voided, `b2` replaces it as the new best block, with state `FAILED`. After the reorg, a `tx2` arrives and since `tx2`'s best block is `b2`, its state would also be `FAILED`:

```
   b2   tx1   tx2
---|-----|-----|---> time
```

Since `tx1`'s best block is still `b1`, even though it's voided, its state is still `ACTIVE`. This would mean that we have two consecutive, valid transactions with retreating feature states, the equivalent to:

- `tx2.timestamp >= tx1.timestamp` and
- `not (f(tx2).feature_state >= f(tx1).feature_state)`

Which breaks premise #3.

We then add boundary heights to the solution. We know that feature states can only change in boundary heights, and then stay the same for the whole evaluation interval. We also know that whenever a new tx arrives it falls in some evaluation interval, between boundary heights `h1` and `h2`. At that time, it's guaranteed that there will be a block at `h1`, and not at `h2`. Also, it holds that `b1.height >= h1.height`, where `b1` is `tx1`'s best block:

```
   h1    b1   tx1
---|-----|-----|---> time
```

Notice that `h2` is omitted, as there's no block at that height yet (it would eventually appear after `tx1`).


This time, instead of associating transaction feature states using best blocks, we'll use the boundary blocks. We can easily retrieve the boundary block that is the closest to a transaction's best block (to the left). In case of `tx1`, that would be `h1`. In other words, `f(tx1) = h1`.

Similarly to the previous example, let's say there's a reorg that voids both `h1` and `b1`, so the new best block is `b3`. We've used the `~` notation to represent a voided block:

```
  ~h1   ~b1   tx1    b2   tx2    h1'   b3
---|-----|-----|-----|-----|-----|-----|---> time
```

This reorg also added the `b2` and `h1'` blocks, which are part of the new blockchain. The height of `h1'` is the same as `h1`, but its timestamp is greater.

To calculate `tx2`'s feature state, we'll get its best block (`b2`) and then its closest boundary block, which has some height lower then `b2`. Let's call it `h0`, as it's the boundary block directly before `h1/h1'`:

```
   h0   ~h1   ~b1   tx1    b2   tx2    h1'   b3
---|-----|-----|-----|-----|-----|-----|-----|---> time
```

From that, it follows that `f(tx2).height < f(tx1).height`, which directly contradicts `f(tx2).feature_state >= f(tx1).feature_state`, breaking premise #3.

Observing that last timeline, we can finally arrive at the working solution. Instead of using the boundary block that is closest to the transaction's best block, we will use the boundary block before that. Let's reproduce the timeline at the beginning of the example:

```
   h0    h1    b1   tx1
---|-----|-----|-----|---> time
```

Now, we've included `h0`, and we'll define `tx1`'s feature state as the same as `h0`, or `f(tx1) = h0`. Reproducing the reorg:

```
   h0   ~h1   ~b1   tx1    b2   tx2    h1'   b3
---|-----|-----|-----|-----|-----|-----|-----|---> time
```

It now follows that `tx2`'s feature state is also the same as `h0`, making `f(tx2) = f(tx1) = h0`. Using this strategy, premise #3's requirements are hold.

## Completing timeline scenarios

In the guide-level section, we said that there are only two possible cases for positioning a transaction and its best block in a timeline. This was in fact incomplete, so let's address this now. Here's a new timeline:

```
   h0    h1   tx1
---|-----|-----|---> time
```

Let's analyze where `tx1`'s best block, `b1`, could be. The following set of intervals covers the whole timeline:

1. `b1 < h0`
2. `h0 <= b1 < h1`
3. `h1 <= b1 <= tx1`
4. `tx1 < b1`

Scenarios #2 and #3 are actually the cases described in the guide-level section, so no need to detail them.

Scenario #1 means that there are no blocks between `h0` and `h1`, which is an evaluation interval. That would mean at least two weeks without any blocks in the network, which is virtually impossible, so this scenario is considered invalid.

Lastly, scenario #4 means that calling `TransactionStorage.get_best_block_tips(tx1.timestamp)` would return a block with `b1.timestamp > tx1.timestamp`. In other words, it would return a block from the future, which does not make sense for this method. Therefore, this scenario is also considered invalid. TODO: Is this true? Should we add an `assert` for that in `get_best_block_tips()`?

## Dealing with reorgs

Let's detail a bit further what could possibly happen when a reorg occurs, and how to deal with them in the context of transactions. There are only two possibilities that affect the Feature Activation process.

### Changes in the best block height

When a reorg occurs, the best block height could change, shifting the transaction's best block between the different timeline intervals described above. The interpretation of what happens in each possible shift is direct from the previous analysis.

The new best block can only shift FROM scenarios #2 and #3 and TO scenarios #2 and #3, as any other shift combination would result in an invalid scenario.

This means that even though a transaction's best block can change, `f(tx1)` will always return the block at `h0`, the same it returned before the reorg. It doesn't change over time, and it doesn't change when reorgs occur. In other words, no explicit handling of reorgs is necessary.

### Changes in the feature state of the best block

When a reorg occurs, another possibility is that the feature state of the best block would change, for example if the original best chain signaled support and enabled some feature, and the new best chain does not.

As described before, the best block can only shift TO and FROM scenarios #2 and #3, meaning that it can only affect the feature state at boundary blocks `h1` and a future `h2`. Since `f(tx1)` always returns `h0`, the transaction's feature state is not affected by the reorg. Again, no explicit handling of reorgs is necessary.

### TODO: Am I missing any other reorg consequence that should be considered?

## Retrieving feature states for Transactions

Given the explanations in the sections above, a new method will be added to the `FeatureService`. Here's its reference implementation:

```python
def get_state_for_transaction(self, *, tx: Transaction, feature: Feature) -> FeatureState:
    best_block_tips = self._tx_storage.get_best_block_tips(tx.timestamp)
    assert len(best_block_tips) >= 1

    best_block_hash = best_block_tips[0]
    best_block = self._tx_storage.get_transaction(best_block_hash)
    assert isinstance(best_block, Block)

    best_block_height = best_block.get_height()
    offset_to_boundary = best_block_height % self._feature_settings.evaluation_interval
    closest_boundary_height = best_block_height - offset_to_boundary
    state_block_height = closest_boundary_height - self._feature_settings.evaluation_interval
    state_block = self._tx_storage.get_transaction_by_height(max(state_block_height, 0))
    assert isinstance(state_block, Block)

    return self.get_state_for_block(block=state_block, feature=feature)
```

That's all its necessary to enable Feature Activation for transactions.

## Drawbacks

### The two-week delay

A drawback is that feature states for transactions will always have a two-week delay when compared to feature states for blocks, making the transaction process a bit longer. The tradeoff is worth it as this makes for a very simple implementation that naturally doesn't need to account for reorgs.

Special care should be taken when designing new features that will affect both blocks and transactions at the same time, as the feature may become `ACTIVE` for blocks before it does for transactions. It's not clear at this point if this will ever be a possibility for a new feature and if it would be a problem, but if it is, I believe we can solve it by introducing a two-week delay on the feature state for blocks, making their states synchronized with transactions.

# Rationale and alternatives
[Rationale and alternatives]: #rationale-and-alternatives

The rationale is explained in the guide-level section above.

TODO: add alternatives

# Prior art
[prior-art]: #prior-art

I couldn't find any prior art relevant to this RFC, considering Hathor's unique DAG architecture for transactions. While the Feature Activation for blocks is heavily based on the Bitcoin solution, the transactions solution is very specific to Hathor.

# Task breakdown

TODO
