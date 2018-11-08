- Feature Name: sub-tokens
- Start Date: 2018-11-07
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

Allow the issuance of other tokens inside Hathor's network. So, many tokens would co-exist inside the same DAG of verifications.

# Motivation
[motivation]: #motivation

The motivation is to increase the usability of the network. When anyone may easily issue their own token, they don't need to bother about all details behind maintening a network. They just focus on their application.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Other tokens inside Hathor's platform will be called sub-tokens.

A sub-token may be issue through a special transaction. This issuance will have Hathor's tokens as inputs, and a new sub-token in its outputs. The settings of this new token will be set during the issuance.

The Hathor's tokens in the input will be locked while the token exists. It is like an "issuance price". A maximum factor of 1,000 will be allowed, i.e., to issue 10,000,000 of a sub-token, it will be required at least 10,000 Hathor's tokens in the input.

Each sub-token will receive a 256-bit number as an unique identifier (uid). It is possible to set user-friendly alias to these uids, but it is configured in each node/wallet. Users should never rely on alias, the the uid of the token must all be checked.

After a sub-token has been created, the new tokens may be freely exchanged exactly like Hathor's tokens. Thus, they will have the same features, like Nano Contracts, timelock, and so on.

It will be possible to create transactions with more than one token, i.e., Hathor's tokens and a sub-token, as long the verification passes for each token.

## Sub-token Settings

- Allowance to issue more sub-tokens like this in the future? Boolean.
- Allowance to destroy sub-tokens in exchange for Hathor's tokens? Boolean.
- Number of precision digits


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Transaction's inputs and outputs will have to differentiate between tokens. Some changes must be made:

1. The sum of inputs must equal the sum of outputs for each token uid (except on the token-issuance transaction).
2. A new type of transaction will exist: `MULTI_TOKEN_TRANSACTION`.
3. Transactions will have a list of token uids inside it. Hathor is always index 0, while the others start at index 1.
4. Each output must have the index of its correspondent token uid in the list of token uids.
5. There will a limit of 16 different tokens in the same transaction (15 sub-tokens + Hathor). Thus, the number of sub-tokens will require 4 bits.

Besides these changes, everything else remains the same. We just have to detail a transaction issuing a sub-token.

The transaction's type header will be of type `TYPE_TOKEN_ISSUANCE` (0x02). It will force the inputs to be all Hathor's tokens, the first output must be the new sub-token being created, and the remaining outputs must be changes in Hathor's tokens.

The script of the new sub-token's output is a regular script, usually a P2PKH format.


# Drawbacks
[drawbacks]: #drawbacks

The main drawback is storage usage. Transactions will be bigger, hence they will use more storage.

Another drawback would be an increase in CPU consumption, because the verification algorithm will be more complicated. But it seems to be a minor drawback.


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
