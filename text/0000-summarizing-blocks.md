- Feature Name: summarizing_blocks
- Start Date: 2018-11-06
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)

# Summary
[summary]: #summary

A mechanism for reducing the storage size and size growth of the Hathor network: create a special block type that summarizes all usable transactions/outputs before it, and change the transaction such that only a minimal portion has to be kept after it is summarized.

# Motivation
[motivation]: #motivation

With the increasing usage of the network, all blocks and transactions have to be stored indefinitely in order to maintain the trust on the network. Storage growth is proportional to the size of the blocks and transactions being accepted.

# Detailed design
[detailed-design]: #detailed-design

### A DAG without data

Transaction data is split in 3: data that can be discarded, data that can't be discarded, and the nonce. Hashing must change to 3 stages: hash1: hash of the data that can be discarded, hash2: hash of (the data that can't be discarded + hash1) and hash3, the final hash: hash of (hash2 + nonce). Data that can't be discarded must be kept minimal, and probably comprises of the hash of the parents, and weight.

The block can go through similar adaptations, but since it shouldn't grow in size as much as the DAG, it can stay unchanged.

With the adapted structure, there is a mechanism for building a DAG without a portion of the data, albeit the missing portion has two obvious consequences: important data is lost and important verifications are no longer possible ("important" is used loosely, for now).

### The "summary block"

A special "summary block" is introduced. It's purpose is to summarize all the important data, which is a subset of all the "discardable data", so that unimportant data can be discarded. More specifically, data that is not important is data of spent outputs or transactions (NOTE: needs better description). For instance, the summary block could have a big table of [output, amount], although other type of scripts will require additional data.

This new type of block can be a part of the block-chain, but should have some differentiations. It should have a higher reward than a regular block, mainly because adds more value to the network than a regular block (NOTE: this is just a claim though), but the rate at which summary blocks are allowed should be small, because it is heavy in size and limiting it conserves bandwidth. (NOTE: maybe there are better ways to tune this tradeoff, like making the reward proportional to the amount of data being made discardable).

### Why trust is preserved?

As it stands, the block-chain, as well as the hathor DAG, have trust given to the largest accumulated weight chain. This means that the longer the chain/DAG, the harder it is to forge a DAG, even if only forging a minimal DAG + summary block.

For stability and increased trust, further restrictions can be placed for accepting summary blocks. Such as requiring a minimal block-chain/DAG length (height?) and only accepting summary of deeper transactions.

# Drawbacks
[drawbacks]: #drawbacks

- Allowing data to be discarded can lead to harder to recover bugs.
- New nodes bootstrapping from incomplete data (DAG with dropped data + summary block), cannot make a full verification of the summary. (NOTE: it should be much harder to forge a summary block than to do a double spend)
- Impact of forging is way way higher. A forged block summary can steal all tokens for itself. (NOTE: this is probably the most important drawback)
- This new mechanics may drive human trust down

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design does not change the rate at which data growth, which should still be dominated by the minimal DAG (full DAG minus discardable data).

Alternatives include:

- Making a summary block without an aggregated table and letting letting unimportant data be dropped as needed. The disadvantage is that it makes picking the unimportant blocks harder, specially because a transaction can have multiple outputs, and when only some of them are spent, the whole droppable data is still important.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How exactly it interacts with nano-contracts and sub-tokens?
- What should be the rules for accepting a summary block?
- How are the incentives for propagating data to new peers affected?

# Future possibilities
[future-possibilities]: #future-possibilities

A mechanism for bootstraping a new client directly from a trusted summarizing block may speed-up adoption significantly.
