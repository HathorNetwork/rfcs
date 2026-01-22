- Feature Name: wallet-lib-shared-facades
- Start Date: 2026-01-21
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: tulio@hathor.network

# Summary
[summary]: #summary

The Wallet Lib facades for the Fullnode and the Wallet Service should share a
subset of the most important methods, with the same parameters and output data
format.

This will cause breaking changes that will require a major version update
on the lib.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

At the moment, both facades are similar, but not equal. This causes duplicated
test suites, requires additional time and attention from the developers and
ultimately introduces bugs on the Wallet Lib clients when they need to switch
between facades.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

To achieve this state it's necessary to change the contracts for some methods,
both in the wallets and in their shared interface `IHathorWallet`, which is now
outdated. An [initial investigation](./shared-contract-investigation.md) was made
to understand the necessary changes, and they will be summarized below.

- Change methods from `sync` to `async` on the Wallet Service facade
- Modify `IHathorWallet` return types that currently do not match the fullnode facade
- Create types for the parameters of both facades, facilitating visual inspection from devs
- Decide on `UTXO` and `Tx History` objects that currently differ, and are used by most of the applications
- Decide which subset of methods will consist of the `IHathorWallet`, and leave each facade to have its own complementary methods
- Add methods that are present in both facades but not in the `IHathorWallet`, such as `getAuthorityUtxo()`
- Create a `shared_types.ts` on the root folder, instead of using types from each facade folder

At the end of this project, it should be possible for a hypothetical client
implementation to call methods from a Fullnode or Wallet Service facades
interchangeably during its operation, with no adaptation to their inner workings.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

The only reasons not to do this would be to preserve backwards compatibility with
existing clients, or to dedicate development priority elsewhere.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The facades could be kept separate and a wrapper `WalletWrapper` could be
developed to abstract away their differences. This would allow for shared
integration tests and a simpler developer experience, while the internal work
is postponed for a better time.

The downside of this alternative is having additional work and complexity in
order to avoid a synchronization work that will have to be done eventually.

By not implementing this shared facade, we keep the responsibility on the client
to know what facade is being used and the details of each one separately in the
client code.

# Prior art
[prior-art]: #prior-art

The objetive of the `IHathorWallet` interface was to create a shared contract for
both facades, but without automated testing it was natural to drift away from
a compliance strict enough for a full Typescript validation.

The main proposal of this RFC is to bring this interface up-to-date with the
current state of the wallets, improving it albeit with breaking changes.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The main questions to resolve through this RFC are the actual method signatures,
deciding which properties are mandatory, which will be optional.

A special attention should be paid to the `null` parameter, which is used in
many places as a valid replace for `undefined`. This is

We should consider keeping a `v2` version with minor and patch updates for
critical issues, but going forward we would fully migrate to `v3`.

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about breaking the facade files into multiple pieces with common
group responsabilities, that could be shared between the two facades. This would
make it easier for LLMs to interact with those files, instead of having
two facades with over 3k lines that do not fit their context windows.

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
