- Feature Name: consensus
- Start Date: 2019-08-06
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

The consensus algorithm is a core module of Hathor and decides which transactions and blocks are executed and which are voided.

# Motivation
[motivation]: #motivation

The transactions are arranged in a DAG, while the blocks are in a Blockchain. They are synced using the p2p network, and the peers must agree on which transactions and blocks are executed (and which are voided).

The consensus cannot depend on the order that the transactions are received by the peer. It is a major requirement since we cannot control the propagation of the transaction in the p2p network. In other words, if two peers have the same data, they must reach the same consensus.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A transaction is executed when the funds have been transferred from the inputs to the outputs. On the other hand, a transaction is voided when the funds have not been transfered.

Two or more transactions will be part of a conflict resolution if they conflict. Basically, a conflict happens when they are trying to spend the same output (i.e., the same funds). In this case, we say that they are double spending the output. The aftermath of a conflict resultion is that all transactions but the winner will be voided. In some cases, all transactions will be voided, which means there may be no winner.

For blocks, they will be voided if they are not in the best blockchain. The best blockchain is the blockchain that has the highest work. Sometimes we refer to it as the longest blockchain.

We say that transactions and blocks verifies their parents. We do not use the verb "to confirm" to prevent misunderstanding with the informal understanding of confirmation. The criteria to say that a transaction is confirmed depends on the receiver. One may say that a transaction is confirmed after being verified by at least one block. Others may say that at least six confirmation are required.

If a transaction is voided, all transactions verifying it are voided as well. In other words, the voided status propagates forward in the DAG.

For further details about the DAG, the blockchain, and the consensus algorithm, see the [Hathor Consensus Algorithm] paper.

## Twin transactions

When two or more transactions have the same inputs and outputs, but have different hashes, they are called twin transactions. Basically, they move the funds from the same sources to the same destinations, but something has changed their hashes. Usually, they will have different parents, but there may be other possible causes.

We do not consider twin transactions an attempt of double spending.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When a new transaction or block arrives, it is desirable to validate it and add it to the DAG in constant time (O(1)).

Both transactions and blocks validation depend only on themselves, their parents, and their inputs; and it already happens in constant time. For further information, see the [Anatomy of a Transaction RFC].

The addition of transactions and blocks to the DAG are different and will be detailed in different sections.


## When a new transaction arrives

After the validation, we already ensure that all parents and inputs already exist. So, we just need to update the metadata of the transactions.

The following metadata is kept for each transaction and is used by the consensus algorithm:

- `spent_outputs`: which outputs have already been spent by which transactions (one or more).
- `conflict_with`: which transactions have conflict with this transaction.
- `voided_by`: which transactions are the cause of voidance of this transaction. If this transaction has at least one conflict, it is also a member `voided_by`.
- `children`: which transactions verify this transaction.
- `first_block`: the first block in the best blockchain that verifies this transaction.
- `accumulated_weight`: the accumulated weight of this transaction (this may not be updated).

The following steps are done when adding a new transaction `A` to the DAG:

1. Update the `spent_outputs` of all `A`'s inputs. During this step, the full-node also checks whether `A` has conflicts.
1. Set the `voided_by` of `A` to the union of the `voided_by` of its parents and inputs.
1. Update the `conflict_with` of `A` and all transactions it has conflict. If there is at least one conflict, `A` adds itself to its `voided_by` metadata.
1. If `voided_by` is not empty, resolve the conflict.

The conflict resolution may be the slowest part of adding a new transaction to the DAG, and will be described later.

The addition of a transaction to the DAG happens in constant time if `voided_by` is empty.


## When a new block arrives

When a new block arrives, we need to update its metadata.

The following metadata is kept for each block and is used by the consensus algorithm:

- `spent_outputs`: which outputs have already been spent by which transactions (one or more).
- `voided_by`: which transactions are the cause of voidance of this block.
- `children`: which blocks verify this block.
- `score`: the amount of the work done from genesis to this block.

The following steps are executed when adding a new block `B` to the DAG:

1. Set `spent_outputs`, `children`, and `voided_by` to empty.
1. Update its score using the score of its parent block and its own weight.
1. If `B`'s parent block is the head of the best blockchain, then `B` is the new head of the best blockchain.


# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
