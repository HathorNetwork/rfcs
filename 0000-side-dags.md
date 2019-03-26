- Feature Name: side-dags
- Start Date: 2019-03-22
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

Allow others to cheaply run their own DAG of transactions, applying with their own rules, and have these transactions verified by Hathor's Main DAG.

# Motivation
[motivation]: #motivation

The motivation is to allow Hathor to scale and meet different necessities. It would allow anyone to cheaply run their own DAGs, which comply with their specific business needs, without any modification in Hathor's source-code.

Other DAGs will have their transactions verified by Hathor Main DAG's blocks and transactions, i.e., external DAGs will increase their transactions' accumulated weight through blocks and transactions from Hathor's Main DAG.

It is interesting that Hathor Main DAG won't have to know other DAG's rules, and, even so, it will verify their transactions. This allows the creation of private DAGs, which will contain private transactions that may be safery audited by a third-party, or the creation of temporary DAGs, that will support a given operation and will be discarded later.

It also works as a sharding strategy, because Hathor's Main DAG won't have the transactions of these side DAGs, reducing the storage requirements.

It also allows an independent network, with many peers and its own DAG, to have only one or two full nodes connected to the Hathor's Main DAG and generating these special transactions to confirm the side DAG's transactions. This network may be private or not.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We have to create a new type of transaction. This transaction will have two parents connected to the Hathor's Main DAG, and two other parents to connected to the side DAG. Then, there are two possible cases when a full node is processing this transaction:

1. If the full node does not have the side DAG, then it always accepts the special transaction, which will have its accumulated weight growing as new blocks and transactions arrive in Hathor's Main DAG.

1. If the full node have a copy of the side DAG, then it applies the rules of this side DAG and verifies whether it must be voided or not. It is interesting that this special transaction will propagate its accumulated weight from the Hathor's Main DAG to the side DAG.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add a new extension of `BaseTransaction` called `ExternalVerification`, which has two new attributes: `side_genesis: bytes` and `side_parents: List[bytes]`. These attributes will work as pointers to transactions of a side-DAG. We have no information about either the side-DAG or the transactions it points to.

When verifying an `ExternalVerification` vertex of the DAG, we will only verify the `side_parents` when the `side_genesis` exists. Otherwise, we just skip them. This will allow a full-node to decide which side-DAGs it will by syncing. All full-nodes must have Hathor's Main DAG.

When calculating the `meta.acc_weight` of a transaction in a side-DAG, we must go through the children in the Main DAG. This will propagate the proof-of-work from Hathor's Main DAG into the side-DAG. Thus, making the transactions of the side-DAG immutable just like a transaction in the Main DAG.

`ExternalVerfication` may have inputs and outputs to pay a fee (which may go to the miners or simply melt some HTRs).


# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives


# Prior art
[prior-art]: #prior-art


# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How to create a transaction with inputs from two different side DAGs?
- Hathor's Main DAG has three genesis transactions. Would `side_genesis` force side-DAGs to have one, and only one, genesis?


# Future possibilities
[future-possibilities]: #future-possibilities


# References
[references]: #references

