- Start Date: 2019-07-24

# Disclosure
[disclosure]: #disclosure

This whole document is an adaptation of the [rust-rfc-process].

# Summary
[summary]: #summary

The "RFC" (request for comments) process is intended to provide a consistent and
controlled path for new features to enter the platform and standard libraries,
so that all stakeholders can be confident about the direction the platform is
evolving in.

# Motivation
[motivation]: #motivation

The freewheeling way that we add new features to Hathor has been good for early
development, but for Hathor to become a mature platform we need to develop some
more self-discipline when it comes to changing the system.  This is a proposal
for a more principled RFC process to make it a more integral part of the overall
development process, and one that is followed consistently to introduce features
to Hathor.

# Detailed design
[detailed-design]: #detailed-design

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a
bit of a design process and produce a consensus among the Hathor community and
the [core team].

## When you need to follow this process
[when-you-need-to-follow-this-process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to
the Hathor distribution. What constitutes a "substantial" change is evolving
based on community norms, but may include the following.

- Any change that affects the transaction or block's serialization format.
- Changes to the validation of transactions and blocks.
- Changes to the opcodes of the script language.
- Changes that affects the consensus of the network, i.e., which transactions
  are voided and which are executed.
- In general any feature that requires a soft or hard fork.

Some changes do not require an RFC:

- Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not
  change meaning".
- Additions that strictly improve objective, numerical quality criteria (warning
  removal, speedup, better platform coverage, more parallelism, trap more
  errors, etc.)
- Additions only likely to be _noticed by_ other developers-of-hathor, invisible
  to users-of-hathor.

If you submit a pull request to implement a new feature without going through
the RFC process, it may be closed with a polite request to submit an RFC first.

## What the process is
[what-the-process-is]: #what-the-process-is

In short, to get a major feature added to Hathor, one must first get the RFC
merged into the RFC repo as a markdown file. At that point the RFC is 'active'
and may be implemented with the goal of eventual inclusion into Hathor.

- Fork the RFC repo https://github.com/HathorNetwork/rfcs
- Copy `0000-template.md` to `text/0000-my-feature.md` (where 'my-feature' is
  descriptive. don't assign an RFC number yet).
- Fill in the RFC
- Submit a pull request. The pull request is the time to get review of the
  design from the larger community.
- Build consensus and integrate feedback. RFCs that have broad support are much
  more likely to make progress than those that don't receive any comments.

Eventually, somebody on the [core team] will either accept the RFC by merging
the pull request, at which point the RFC is 'active', or reject it by closing
the pull request.

Whomever merges the RFC should do the following:

- Assign an id, using the PR number of the RFC pull request. (If the RFC has
  multiple pull requests associated with it, choose one PR number, preferably
  the minimal one.)
- Add the file in the `text/` directory.
- Create a corresponding issue on
  [Hathor repo](https://github.com/HathorNetwork/hathor-python)
- Fill in the remaining metadata in the RFC header, including links for the
  original pull request(s) and the newly created Hathor issue.
- Commit everything.

Once an RFC becomes active then authors may implement it and submit the feature
as a pull request to the Hathor repo. An 'active' is not a rubber stamp, and in
particular still does not mean the feature will ultimately be merged; it does
mean that in principle all the major stakeholders have agreed to the feature and
are amenable to merging it.

Modifications to active RFC's can be done in followup PR's. An RFC that makes it
through the entire process to implementation is considered 'complete' and is
removed from the [Active RFC List]; an RFC that fails after becoming active is
'inactive' and moves to the 'inactive' folder.

# Alternatives
[alternatives]: #alternatives

Retain the current informal RFC process. The newly proposed RFC process is
designed to improve over the informal process in the following ways:

- Discourage unactionable or vague RFCs
- Ensure that all serious RFCs are considered equally
- Give confidence to those with a stake in Hathor's development that they
  understand why new features are being merged

As an alternative alternative, we could adopt an even stricter RFC process than
the one proposed here. If desired, we should likely look to Python's [PEP]
process for inspiration.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Does this RFC strike a favorable balance between formality and agility?
2. Does this RFC successfully address the aforementioned issues with the current
   informal RFC process?
3. Should we retain rejected RFCs in the archive?

[core team]: https://hathor.network/team/
[PEP]: http://legacy.python.org/dev/peps/pep-0001/
[rust-rfc-process]: https://github.com/rust-lang/rfcs/blob/master/text/0002-rfc-process.md
