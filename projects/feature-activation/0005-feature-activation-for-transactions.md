- Feature Name: Feature Activation for Transactions
- Start Date: 2023-09-11
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

This problem demonstrated to be harder than believed in the beginning of this work, resulting in multiple completely different ideas being explored. Some of those became RFCs by themselves, now kept just for reference in an [iterations directory](./0005-iterations). Eventually we would find problems in each solution, and only by leveraging those initial iterations we arrived at the proposal in this document. The other ideas were convoluted and complex, generating multiple edge cases that were hard to keep track and error-prone. Then, the idea is to go back to the base problem and find a simpler solution. Considering that, some of the text was extracted from the previous documents.

To use block features states to retrieve feature states for transactions, let's first consider the possible state values. Blocks use multiple states to represent the Feature Activation process, such as `STARTED`, `MUST_SIGNAL`, `LOCKED_IN`, `ACTIVE`, etc, as they're responsible for the whole logic of the system. In the context of transactions, contrary to blocks, not all these states are relevant. In fact, for a transaction, it only matters if a feature is either `ACTIVE` or `INACTIVE`. For brevity, from this point forward when we say that a block is `ACTIVE`, we mean that its state for a certain feature is `ACTIVE`, and when we say that a block is `INACTIVE`, we mean that its state for a certain feature is any state _but_ `ACTIVE`.

To determine the state of a feature for a Transaction, the most obvious idea is to get the state of the current best block. This is problematic when reorgs happen, as detailed in the previous documents. Another previous idea was adding a two-week delay (one Evaluation Interval) between the process for blocks and transactions, by using the state of the boundary block from the evaluation interval _before_ the current evaluation interval. This creates a "buffer" period that helps make sure a feature is consolidated as `ACTIVE` before enabling it for transactions. This idea is good, because this buffer makes so that only extremely large (and therefore unlikely) reorgs would change the state of a transaction. The problem was how this buffer was defined in [iteration 1](./0005-iterations/iteration1.md), which used a similar idea. There, the reorg problem persisted.

With that in mind and going back all the way to the motivation of BIP 8 vs BIP 9 (as discussed in the original RFC defining Feature Activation for Blocks), we recall that block heights were used instead of block timestamps, mainly to prevent unexpected behavior when the hash rate is too volatile. However, using timestamps makes for a pretty straightforward solution for transactions. Combining that with the buffer idea, we can guarantee that an unwanted scenario is unlikely enough to consider it unrecoverable, forcing manual intervention. This buffer period can be as long as we want: the longer it is, the more unlikely is a manual intervention, but the delay for activating a feature for transactions increases. This tradeoff will be explored in sections below.

Let's now define the solution: **for a feature, a transaction is considered `ACTIVE` if its timestamp is greater than one Evaluation Interval after the _first_ `ACTIVE` block's timestamp**. This will be detailed in the Reference-level section. The fact that timestamps are "mutable" and not signed will also be taken into account.

This is essentially very similar to what was defined in the first iteration, that can be rewritten as "a transaction is `ACTIVE` if the boundary block of the previous evaluation interval is `ACTIVE`". However, a small reorg caused problems there, as it could shift the closest boundary block from before to after the transaction (time-wise), making the transaction state retreat from `ACTIVE` to `INACTIVE`. By using the `block height <-> timestamp` duality, this problem is resolved. In other words, a specific block height _can_ be shifted around the timestamp of transaction, but a specific timestamp _cannot_ — so we define the tipping point to activate transactions as a timestamp, rather than a block height.

Similarly, a large reorg could also cause problems if it was large enough so that the _first_ `ACTIVE` block for a feature participates in the reorg. Instead of preventing a large reorg in terms of how many blocks were reorged, we will again leverage the `block height <-> timestamp` duality and prevent a large reorg in terms of time. If we've already passed one Evaluation Interval after the _first_ `ACTIVE` boundary block, we will have `ACTIVE` transactions. Therefore, we cannot allow reorgs that could render the feature `INACTIVE`. That would only happen if the reorg includes that _first_ `ACTIVE` block.

Summing up: **if we've passed one Evaluation Interval after the _first_ `ACTIVE` boundary block AND the reorg includes that _first_ `ACTIVE` boundary block, we cannot recover**. That would require revalidation of transactions. Instead, since this scenario is extremely unlikely, we error out and exit the full node, requiring manual intervention.

Equipped with only those two definitions, the solution is complete. No other edge cases have to be handled. Also, to be clear, no change in the Feature Activation for Blocks is necessary.

# Reference-level explanation
[Reference-level explanation]: #reference-level-explanation

In this section, technical details are expanded for what was described above. Before detailing the solution, let's formalize the context.

## Overview

### Premises

1. After an `ACTIVE` block, all next blocks will also be `ACTIVE` (considering the same blockchain).
2. Feature states for transactions are only `ACTIVE` or `INACTIVE`.

### Requirements

Using the premises above, we must define a function that returns a state, given a transaction (note: we actually return multiple states for multiple features). That function must satisfy the following requirements:

1. All transactions that are received after (time-wise) an `ACTIVE` block in the best chain, must also be `ACTIVE`.
2. When all transactions are ordered by timestamp, there must not be an `INACTIVE` transaction after an `ACTIVE` transaction. In other words, as soon as one transaction becomes `ACTIVE`, every future transaction will also be `ACTIVE`.

### Definitions

Repeated here from the Guide-level section.

#### State of a transaction

For a feature, a transaction is considered `ACTIVE` if its timestamp is greater than one Evaluation Interval after the _first_ `ACTIVE` block's timestamp.

#### Dealing with reorgs

If we've passed one Evaluation Interval after the _first_ `ACTIVE` boundary block AND the reorg includes that _first_ `ACTIVE` boundary block, the reorg is considered invalid and the full node halts execution.

## Retrieving Feature States for Transactions

Analogously to the Feature Activation for Blocks, the state for a transaction will be retrieved from a method in the `FeatureService`. Considering the definition above, a pseudo reference-implementation is provided:

```python
class FeatureService:
    def is_feature_active_for_transaction(self, *, transaction: Transaction, feature: Feature) -> bool:
        first_active_block = self._get_first_active_block(feature)

        if not first_active_block:
            return False

        # equivalent to two weeks
        avg_time_between_boundaries = self._feature_settings.evaluation_interval * settings.AVG_TIME_BETWEEN_BLOCKS
        activation_threshold = first_active_block.timestamp + avg_time_between_boundaries
        # We also use the MAX_FUTURE_TIMESTAMP_ALLOWED to take into account that we can receive a tx from the future
        is_active = transaction.timestamp > activation_threshold + settings.MAX_FUTURE_TIMESTAMP_ALLOWED

        return is_active

    def _get_first_active_block(self, feature: Feature) -> Optional[Block]:
      """
      Return the first ever block that became ACTIVE for a specific feature (always a boundary block),
      or None if this feature is not ACTIVE.

      This can be implemented by recursively hopping boundary blocks until
      we find a block that is ACTIVE and has a parent that is LOCKED_IN.
      """
      raise NotImplementedError
```

## Dealing with reorgs

Considering the definition above, a function must be called in the `ConsensusAlgorithm.update()` method that determines whether a reorg is invalid. If it is, the full node will **halt execution**.

We can consider a reorg invalid if the current time is greater than the common block's timestamp plus one Evaluation Interval. This is a superset of reorgs that include the definition and simplifies implementation, and is also extremely unlikely. A pseudo reference-implementation is also provided:

```python
def reorg_is_invalid(common_block: Block) -> bool:
    now = self.reactor.seconds()
    # equivalent to two weeks
    avg_time_between_boundaries = settings.FEATURE_ACTIVATION.evaluation_interval * settings.AVG_TIME_BETWEEN_BLOCKS
    # We also use the MAX_FUTURE_TIMESTAMP_ALLOWED to take into account that we can receive a tx from the future.
    # This is redundant considering we also use it in is_feature_active_for_transaction(), but we do it here too to restrict reorgs even further.
    is_invalid = now >= common_block.timestamp + avg_time_between_boundaries - settings.MAX_FUTURE_TIMESTAMP_ALLOWED
    
    return is_invalid
```

This means that if a reorg voids at least two-weeks worth of blocks, it's possible that the reorg includes a first `ACTIVE` block AND the respective transaction activation threshold. This situation cannot be recovered, as there could be `ACTIVE` transactions that would not be re-validated. Therefore, the full node errors and exits, requiring manual intervention.

Full node operators must remove the storage and perform a re-sync from a snapshot before the reorg (or a sync from scratch), so the full node doesn't experience this reorg, and operation would resume normally. This scenario is extremely unlikely (this will be explored further down).

## Discussion and examples

In this section, we will explore some illustrated examples in an effort to improve clarity and try to prove the solution works. We will first explore a similar idea using block heights instead of timestamps, and only by understanding its issue, we'll introduce timestamps as a solution. Therefore, the first examples will be very similar to the ones in the first iteration document.

### Analogous solution using block heights

Here, the proposed solution would be: "a transaction is `ACTIVE` if the boundary block of the current best block's previous evaluation interval is `ACTIVE`". Also, the proposed solution for dealing with reorgs: "a reorg is invalid if it reorgs more than 40.320 blocks (one Evaluation interval)".

We will define a timeline and put some blocks and transactions in it:

```
   E0    E1    BB   tx1
---|-----|-----|-----|---> time
   ac    ac    ac    ac
```

On top of the timeline, we see vertex names. Below it, we see feature states for the respective vertex (`ac` stands for `ACTIVE`). Blocks `E0` and `E1` are evaluation boundaries, meaning that there is one Evaluation Interval between them (40.320 blocks, equivalent to two weeks). Then, `tx1` is a transaction that arrived when `BB` was the current best block. The next evaluation boundary, `E2`, has not yet been reached, so it's not shown.

To determine the feature state for `tx1`, we first get the current best block (`BB`), then its closest evaluation boundary (`E1`), then the _previous_ evaluation boundary (`E0`). This is all easily calculated by using evaluation interval math. Then, it follows that `tx1`'s state is `ACTIVE`.

#### Dealing with large reorgs

What would happen if a reorg of more than one Evaluation Interval occurs? It would be possible that `E0` is included in the reorg, which would mean the new `E0` (let's call it `E0'`) could be `FAILED` instead of `ACTIVE`, for example (let's call it `in` for `INACTIVE`). The new timeline would be:

```
before:
   E0    E1    BB   tx1
---|-----|-----|-----|---> time
   ac    ac    ac    ac

after:
   E0'   E1'   BB'  tx1
---|-----|-----|-----|---> time
   in    in    in    ac
```

Since transactions will NOT be re-validated, `tx1` would remain `ACTIVE` even though it shouldn't anymore. Therefore, this reorg cannot be allowed. It is invalid and would halt full node execution, because it reorged more than 40.320 blocks. For the size of this reorg, it is extremely unlikely, so this halting would be extremely rare.

If the reorg is smaller than that, it's guaranteed that `E0` would NOT participate in the reorg, and then the feature would remain `ACTIVE`. Here's a timeline for such reorg:

```
before:
   E0    E1    BB   tx1
---|-----|-----|-----|---> time
   ac    ac    ac    ac

after:
   E0    E1'   BB'   tx1
---|-----|-----|-----|---> time
   ac    ac    ac    ac
```

Blocks `E1` and `BB` are changed, but `E0` is not, so the feature remains `ACTIVE`, and so `tx1`, correctly. From this analysis, it looks like large reorgs that could generate an invalid state are dealt with, and that small reorgs cannot generate an invalid state. Let's examine that below.

#### Dealing with small reorgs

From the previous analysis, it looks like the solution could work. However, we ended up finding a problem with it. Let's suppose a small reorg of 5 blocks, which is plausible. It's possible that the new best chain arranges blocks in such a way that blocks are shifted in time. Below is a perfectly valid small reorg, where we also added a new `tx2` that arrives after the reorg, and `bb` as the new best block observed by `tx1` and `tx2`:

```
before:
   E0    E1    BB   tx1
---|-----|-----|-----|---------------> time
   ac    ac    ac    ac

after:
   E0          bb   tx1   tx2    E1'   BB'
---|-----------|-----|-----|-----|-----|---> time
   ac          ac    ac    in    ac    ac
```

To determine the feature state for `tx2`, we first get its observed best block (`bb`), then its closest evaluation boundary (`E0`), then the _previous_ evaluation boundary, `E(-1)`, which is not shown. `E(-1)` could be `INACTIVE`, rendering `tx2` as `INACTIVE` too. Considering that `tx1`'s state is never recalculated, we would end up with an `ACTIVE` transaction followed by an `INACTIVE` transaction, which is invalid.

Therefore, we conclude that this solution is flawed. Let's try to fix this issue by introducing the use of timestamps.

### Fixing the issue by using timestamps

Using the actual solution defined in this document, let's revisit the examples above. Here's the initial timeline:

```
   E0   tE1   tx1
---|-----•-----|---> time
   ac          ac
```

Now, instead of using the `E1` block, we use `tE1`, which is just a timestamp and not an actual block. It's calculated as `tE1 = E0.timestamp + 2 weeks (one Evaluation Interval)`, and can be interpreted as the "expected timestamp of `E1`". Per definition, the state of `tx1` is `ACTIVE` if `tx1.timestamp > tE1`, which is true.

Here, we'll not consider the `MAX_FUTURE_TIMESTAMP_ALLOWED` (like in the reference implementation), for simplicity. It's as `MAX_FUTURE_TIMESTAMP_ALLOWED = 0`, that is, if a transaction exists, it's guaranteed that `current_time >= tx.timestamp`.

#### Dealing with large reorgs

What would happen if a reorg of more than one Evaluation Interval occurs? It would be possible that `E0` is included in the reorg, which would mean the new `E0` could be `FAILED` instead of `ACTIVE`, for example. The new timeline would be:

```
before:
   E0   tE1   tx1
---|-----•-----|---> time
   ac          ac

after:
   E0'  tE1'  tx1
---|-----•-----|---> time
   in          ac
```

Since transactions will NOT be re-validated, `tx1` would remain `ACTIVE` even though it shouldn't anymore. Therefore, this reorg cannot be allowed. It is invalid and would halt full node execution, because it reorged more than two-weeks worth of blocks. For the size of this reorg, it is extremely unlikely, so this halting would be extremely rare. This is all completely analogous to the previous solution, except a reorg is considered invalid if it reorgs more than two-weeks (in time), instead of more than 40.320 blocks. In other words, the time between the common reorg block and the current time must be less than two weeks.

If the reorg is smaller than that, it is guaranteed that either:

1. `tE1` has been reached by the current time (so there are `ACTIVE` transactions) BUT `E0` did not participate in the reorg, OR
2. `E0` participated in the reorg BUT the current time has not reached `tE1` (so there are no `ACTIVE` transactions).

In other words, this holds true because by definition transactions can only be `ACTIVE` if `current_time > tE1`, and the time distance between `E0` and `tE1` is two weeks, so it's impossible for a reorg smaller than two weeks to include `E0` if there are `ACTIVE` transactions. Now let's look at the small reorgs issue again.

#### Dealing with small reorgs

Let's simulate the same small reorg that was simulated in the previous solution:

```
before:
   E0   tE1   tx1
---|-----•-----|---------------> time
   ac         ac

after:
   E0   tE1   tx1   tx2    E1' 
---|-----•-----|-----|-----|---> time
   ac          ac    ac    ac    ac
```

Now, the actual `E1'` block appeared _after_ the expected `tE1`, but it doesn't affect state calculations.

To determine `tx2`'s state, we check that it satisfies `tx2.timestamp > tE1`, which is true, so `tx2` is `ACTIVE`, differently from the previous solution that resulted in `INACTIVE`. Therefore, the reorg is completely valid.

### Conclusion

It appears that the solution using timestamps works for small reorgs, while large reorgs that would cause issues are prevented by halting full node execution.

An interpretation to aid intuition for comparing both solutions is observing the fact that reorgs can "compress" and "dilate" blocks in relation to time, which ruins the solution using only block heights. By leveraging time instead, the activation threshold is pinned to the timeline such that shifting blocks in relation to time doesn't affect the threshold.

## Likelihood of halting the full node

On this section we'll explore how likely it is for the full node execution to be halted (and require manual intervention), which is a possible but undesired scenario.

The full node will be halted if a reorg affects more than one Evaluation Interval, which is 40.320 blocks, or two weeks considering the average time between blocks of 30 seconds. For reference, in the [wallet-service](https://github.com/HathorNetwork/hathor-wallet-service-sync_daemon/blob/master/src/utils.ts#L67-L73) we have an alert that considers a reorg of more than 30 blocks to be `CRITICAL`, the highest severity possible. Just for the sheer difference in scale to an already unlikely critical reorg, it's clear that a reorg of a full Evaluation Interval is extremely improbable.

We have to consider we're using time differences instead of amount of blocks, but we'll use this as a premise: a reorg of 40.320 blocks is rare enough that it is acceptable to halt full node execution if it happens. Then, considering the time difference of two weeks, we have three possibilities:

1. The block average is respected and we have exactly 40.320 blocks in two weeks. In this case, the two-week reorg is exactly as unlikely as the reorg of 40.320 blocks.
2. The hash rate suddenly increases and we have more than 40.320 blocks in two weeks. This means that the two-week reorg would affect more than 40.320 blocks, which is even more unlikely than the previous case.
3. Lastly, the hash rate suddenly drops and we have less than 40.320 blocks in two weeks. Then, the two-week reorg would affect less than 40.320 blocks, so we'll turn our attention to this case.

If that last case happens, by definition it would also mean that the average time between blocks is higher than 30 seconds. When the amount of blocks in two weeks decreases, the likelihood of a reorg happening increases, but the average time between blocks also increases. That average is defined by `avg = 40.320 * 30 / N` (in seconds), where `N` is the amount of blocks in the two-week interval.

So if for example there are 20.160 blocks in two-weeks, the average time between blocks would be 60 seconds. Here's a table with some other examples:

| N      | avg      |
|--------|----------|
| 40.320 | 30 s     |
| 20.160 | 60 s     |
| 10.000 | ~120 s   |
| 1.000  | ~20 min  |
| 100    | ~200 min |
| 10     | ~30 h    |

As `N` decreases, the probability of a reorg increases, but the `avg` also increases. A reorg of 1.000 blocks, which is already extremely unlikely, would only be possible if we were having only one block every 20 minutes, which is also very unlikely. For a reorg of 100 blocks, which is creeping into the likely territory (but it's still way higher than our alert for critical reorgs), we would only have one block for every ~200 minutes, or ~3 hours. This is almost impossible in theory, considering that our DAA would prevent this increase in average block times.

Then the only real situation where such a reorg would be more or less likely is if we were without miners for most of the two-week interval. At this point, this would be a full-fledged critical incident and maybe it would even be a good thing that full nodes would halt execution and require manual intervention. In practice, given all that, I expect that a full node halting will never actually happen.

## Example - Releasing new Nano Contracts 

To illustrate the usage of Feature Activation for Transactions, we'll demonstrate how it would be used to release a new Nano Contract.

To release a new Nano Contract, a new value would be added to the `TxVersion` enum, allowing deserialization of a different kind of transaction. Since the `FeatureService.is_feature_active_for_transaction()` method requires a `Transaction` instance, it cannot be accessed during deserialization, as the instance hasn't been construct yet. The deserialization will always succeed, even if the Feature fails to become `ACTIVE`.

Then, a new method must be added in the verification phase of `BaseTransaction`, to assert that the deserialized `TxVersion` is valid. In that method, `is_feature_active_for_transaction()` would be called, and depending on the state returned, the `TxVersion` representing the new Nano Contract would be accepted or not.

This demonstrates that there are two possible known use cases for Feature Activation for Transactions:

1. Changing behavior in transaction validation. In this case, the usage is straight forward, simply a call to `FeatureService.is_feature_active_for_transaction()` to check whether the new feature is `ACTIVE`.
2. Changing behavior in transaction deserialization. In this case, it may be necessary to create a new transaction validation method that verifies the validity of using a new deserialization feature.

## Mutability of transaction timestamps

One complication factor that we haven't considered yet is the fact that the timestamp of a transaction can be tempered with, as it's not part of the transaction signature. This means that after a transaction is pushed to the network, a third-party can re-push a copy of that transaction, only changing its timestamp and weight. Then, a third party could manipulate a transaction in such a way that the transaction's feature state is toggled (to/from `ACTIVE` from/to `INACTIVE`), if the third-party transaction has a higher weight than the original transaction.

Let's consider an example. A feature is created to activate a new Nano Contract, that is, a new `TxVersion` will become allowed during deserialization. At timestamp `t`, that feature becomes active for transactions, and a transaction using the new Nano contract is pushed to the network, with timestamp `t+1`. It's perfectly valid, and is accepted by the network. Then, a third-party copies this transaction, increasing its weight and changing its timestamp to `t-1`, and pushes it to the network. That copy would win over the original transaction, voiding it (for its weight), but since from the perspective of the copy the new Nano Contract is not activated yet, that transaction would not even be accepted in the first place. Therefore, the original transaction would remain valid and accepted. An analogous example can be created for the opposite situation (an original `INACTIVE` transaction being shifted by a third-party to become `ACTIVE`).

In other words, if a third-party tries to manipulate a transaction such that it's shifted before or after the activation threshold, the copied transaction will only remain valid if the usage of a feature doesn't affect the validity of that transaction in the first place.

Therefore, the mutability of transaction timestamps is not an issue for the solution proposed in this document.

# Drawbacks
[drawbacks]: #drawbacks

### The two-week delay

A drawback is that feature states for transactions will always have a two-week delay when compared to feature states for blocks, making the transaction process a bit longer. The tradeoff is worth it as this makes for a very simple implementation that naturally doesn't need to account for reorgs.

Special care should be taken when designing new features that will affect both blocks and transactions together, as the feature may become `ACTIVE` for blocks before it does for transactions. It's not clear at this point if this will ever be a possibility for a new feature and whether it would be a problem. For the foreseen use cases (Merkle Path length, SIGHASH, and Nano Contracts) this is not an issue.

# Rationale and alternatives
[Rationale and alternatives]: #rationale-and-alternatives

This RFC passed through a lot of completely different idea iterations before arriving at this solution.

The first idea that was discarded right from the beginning was using signal bits for transactions, instead of leveraging the Feature Activation for Blocks. This would mean a completely separate process for transactions, and a lot of the already implemented requirements would have to be reimplemented. It made much more sense to leverage the existing block mechanism.

Then, a whole other group of ideas was studied, that was trying to find a simple way of choosing an "associated feature block" for each transaction. The feature state of a transaction would simply be the feature state of that associated block. The problem was finding an associated block that makes sense, and that works when reorgs happens, while satisfying the requirement that the state of transactions cannot "retreat" from `ACTIVE` to `INACTIVE` when transactions are ordered by timestamp. We tried a lot of different options, using best blocks, first blocks, first blocks of parents, introducing a delay of an evaluation interval between a tx and its associated block, etc. After a lot of work analysing those options, we would always find a case where the requirements would be broken and the solution became convoluted.

Then, we tried to come up with a solution taking inspiration from the existing Reward Lock mechanism. This seemed to work, but given the differences between the Reward Lock and the Feature Activation, the solution got too complex and with too many edge cases to handle.

Those solutions are available in the [iterations directory](./0005-iterations), in incomplete form. They're kept there just for reference. After discussing those issues and possible new paths, three new different ideas emerged:

1. Revalidate all transactions after a common reorg block, and revert the reorg if any transaction becomes invalid. This would have performance issues, and would also require a refactor in the consensus so that a failed consensus update could be rolled back (similar to a database transaction commit/rollback).
2. Refactor the full node so vertex metadata would be different per chain. Currently, we have one metadata object per vertex that is the same for all chains. It was possible that this would solve issues, but was a sizeable refactor.
3. Use soft checkpoints generated by code to guarantee that an `ACTIVE` feature never reverts back to `INACTIVE`. This would touch some sensitive parts of the code and is complex enough to consider it error-prone to implement, so was discarded. An observation is that this could actually be used in the solution of this document, replacing the full node halting to guarantee a first `ACTIVE` block never changes. If we conclude that halting the full node is too extreme, we could resort to soft checkpoints instead, but I believe the tradeoff of halting is better.

Finally, using all the knowledge gathered from those explorations, we arrived at the solution proposed here.

# Prior art
[prior-art]: #prior-art

I couldn't find any prior art relevant to this RFC, considering Hathor's unique DAG architecture for transactions. While the Feature Activation for blocks is heavily based on the Bitcoin solution, the transactions solution is very specific to Hathor.

# Task breakdown

Here's a table of main tasks:

| Task                                       | Dev days |
|--------------------------------------------|----------|
| Implement state retrieval for transactions | 0.5      |
| Implement handling of large reorgs         | 0.5      |
| Implement tests and simulations            | 2        |
| **Total**                                  | **3**    |
