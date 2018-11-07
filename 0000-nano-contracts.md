- Feature Name: nano_contracts
- Start Date: 2018-11-07
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)

# Summary
[summary]: #summary

Support to very simple Smart Contracts, in which two or more people transfer their tokens to a contract, and the winner takes all tokens. A contract is simply a set of rules applied to decide the final distribution of tokens.

# Motivation
[motivation]: #motivation

The motivation is to allow people to settle contracts using Hathor's network. A contract is a set of rules that can't be changed and will be applied to decide the final distribution of tokens.

With this feature, Hathor may be used to many different application, from betting to hedging.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A Nano Contract is a special output in a transaction. This output can only be spent when a set of rules is valid. This set of rules is flexible and configured by the creators of the Nano Contract. Differently from Ethereum, Nano Contracts are not Turing complete.

There will an external source of truth, which will provide data that will be used to set the final distribution of tokens. This external source of truth is called Oracle. In the future, the Oracle which provides the data that will be used to execute or terminate the contract.

There are multiple applications for nano contracts, such as hedging (options and future contracts) and betting.

Imagine that two people would like to bet 1 token each in which country will win the World Cup. The rules would be: each of them will choose a country and, the winner receives 2 tokens. If both are wrong, each one gets their 1 token back.

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Hathor programmers should *think* about the feature, and how it should impact the way they use Hathor. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Hathor programmers and new Hathor programmers.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The Nano Contract consists of a few special opcodes in the scripting language of the outputs.

## Opcodes

### OP_CHECKSIG_ORACLE

It consumes three parameters from the stack: `<oracleData> <oracleSig> and <oraclePubKey>`. It checks the data and signature against the pubkey. If the signature is valid, it pushes DATA to the stack. Otherwise, it fails.

For example, the following script would check whether the data was signed by the Oracle or not: `OP_DUP OP_HASH160 <oraclePubKeyHash> OP_EQUALVERIFY OP_CHECKSIG_ORACLE`.

The expected input data must be: `<oracleData> <oracleSig> <oraclePubKey>`.

After running, the stack would be: `<oracleData>`.

This should be the first step when creating a nano contract most of the times.


### OP_MATCH_INTERVAL

It consumes a dynamic number of parameters from stack: `<x> <n> <pubkeyHash1> <a1> <pubkeyHash2> <a2> <pubkeyHash3> … <pubkeyHashN> <an> <pubkeyHash(N+1)>`. After running, it pushes only one of the `<pubkeyHash>` to the stack.

After running, the stack would be: `<pubkeyHash>`.

It checks in which interval the number `<x>` belongs to. If it belongs to (-inf, a1], pubkeyHash1 wins, it it belongs to (a1, a2], pubkeyHash2 wins, if it belongs to (a2, a3], pubkeyHash3 wins, and so on, until (a_(n-1), an] and (an, inf).

The winner will be pushed to the stack in the end.


### (IDEA) OP_JSON_GET
It consumes two parameter from the stack: `<query> <json>`. The json must be encoded using utf-8.

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
