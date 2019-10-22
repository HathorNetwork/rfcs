- Feature Name: difficulty_adjustment_algorithm
- Start Date: 2019-10-22
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Jan Segre <jan@hathor.network>

# Summary
[summary]: #summary

Change our current difficulty adjustment algorithm (DAA) to use linearly
weighted moving averages (LWMA) and a better future time limit (FTL).

# Motivation
[motivation]: #motivation

Our current DAA is too slow to react to hashpower changes and could be stuck for
a long time if the hashpower drops suddenly. With a better DAA we could respond
to hashpower changes much faster and potentially mitigate stalling attacks. From
investigating the latest researches on DAAs it seems LWMA is the best candidate.

Adjusting the difficulty faster doesn't seem to prevent or amplify on/off mining
(aka Stalling Attack, aka Difficulty Stranding Attack or when miners turn off
mining for the difficulty to fall and turn on to mine easier and have a higer
profit) but it allows recovering sooner.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Terms and constants:

- DAA: Difficulty Adjustment Algorithm
- LWMA: Linearly Weighted Moving Average
- FTL: Future Time Limit
- N: size of the moving average window
- T: target solvetime

Use a [LWMA DAA][lwma-difficulty-algorithm] with N=134 with an FTL of 300s (10x
our average block time). The DAA is basically:

```
# both difficulties and solvetimes have size N
next_difficulty = average(difficulties) * T / LWMA(solvetimes)
```

Adoption of this algorithm is a breaking change that requires either a network
reset or a fork. Since there isn't a mainnet running yet, a reset is probably
better because it dismisses the need for a legacy DAA for loading old blocks.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

> TODO
>
> This is the technical portion of the RFC. Explain the design in sufficient
> detail that:
>
> - Its interaction with other features is clear.
> - It is reasonably clear how the feature would be implemented.
> - Corner cases are dissected by example.
>
> The section should return to the examples given in the previous section, and
> explain more fully how the detailed proposal makes those examples work.

## LWMA difficulty algorithm
[lwma-difficulty-algorithm]: #lwma-difficulty-algorithm

```python

```

# Drawbacks
[drawbacks]: #drawbacks

A really high hashpower (>90%) drop will still cause a significant recover time,
however if there is a miner with that much hashpower share we have bigger (51%
attack for example) problems to worry about.

We don't really have to do this now, we could do a fork to introduce this change
later, although that would mean we need some legacy code to process pre-fork
blocks.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The DigiShield algorithm seems like a reasonable alternative that has been
  around longer and is used by bigger coins. Although [zawy][zawy] seems to have
  demonstrated that LWMA is superior to DigiShiled.
- [Komodo's APoW][komodo_apow] tackles this problem very directly, but the
  algorithm isn't well defined and the tradeoffs aren't too clear because they
  have a DPoW mechanism (a sort of checkpoint using Bitcoin) that probably
  interferes with the DAA.

[komodo_apow]: https://komodoplatform.com/adaptive-proof-of-work/

> TODO
>
> - Why is this design the best in the space of possible designs?
> - What other designs have been considered and what is the rationale for not
>   choosing them?
> - What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

The proposed algorithm is identical (although reimplemented in Python instead)
to the [LWMA difficulty algorithm][lwma_daa] proposed by [zawy][zawy], which is
now in use by a lot of coins.

[zawy]: https://github.com/zawy12
[lwma_daa]: https://github.com/zawy12/difficulty-algorithms/issues/3

> TODO: more info
>
> Discuss prior art, both the good and the bad, in relation to this proposal.
> A few examples of what this can include are:
>
> - For protocol, network, algorithms and other changes that directly affect the
>   code: Does this feature exist in other blockchains and what experience have
>   their community had?
> - For community proposals: Is this done by some other community and what were
>   their experiences with it?
> - For other teams: What lessons can we learn from what other communities have
>   done here?
> - Papers: Are there any published papers or great posts that discuss this? If
>   you have some relevant papers to refer to, this can serve as a more detailed
>   theoretical background.
>
> This section is intended to encourage you as an author to think about the
> lessons from other blockchains, provide readers of your RFC with a fuller
> picture. If there is no prior art, that is fine - your ideas are interesting to
> us whether they are brand new or if it is an adaptation from other blockchains.
>
> Note that while precedent set by other blockchains is some motivation, it does
> not on its own motivate an RFC. Please also take into consideration that Hathor
> sometimes intentionally diverges from common blockchain features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

> TODO
>
> - What parts of the design do you expect to resolve through the RFC process
>   before this gets merged?
> - What parts of the design do you expect to resolve through the implementation
>   of this feature before stabilization?
> - What related issues do you consider out of scope for this RFC that could be
>   addressed in the future independently of the solution that comes out of this
>   RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

We could incorporate Komodo's APoW rule to allow recovering on one block. This
needs more research on how APoW really works.

> TODO
>
> Think about what the natural extension and evolution of your proposal would be
> and how it would affect the network and project as a whole in a holistic way.
> Try to use this section as a tool to more fully consider all possible
> interactions with the project and network in your proposal. Also consider how
> this all fits into the roadmap for the project and of the relevant sub-team.
>
> This is also a good place to "dump ideas", if they are out of scope for the
> RFC you are writing but otherwise related.
>
> If you have tried and cannot think of any future possibilities,
> you may simply state that you cannot think of anything.
>
> Note that having something written down in the future-possibilities section
> is not a reason to accept the current or a future RFC; such notes should be
> in the section on motivation or rationale in this or subsequent RFCs.
> The section merely provides additional information.
