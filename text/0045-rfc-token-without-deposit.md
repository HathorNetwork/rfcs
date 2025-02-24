- Feature Name: transaction_fee_minting
- Start Date: 2025-02-24
- RFC PR: (to be created)
- Hathor Issue: (leave this empty)
- Author: Raul Soares de Oliveira

# Summary
[summary]: #summary
Currently, when minting X tokens, a [deposit of P% (1%) of X in HTR is required](./0011-token-deposit.md). This deposit can later be withdrawn when the tokens are melted. 

This proposal suggests an alternative mechanism where, instead of requiring an upfront deposit of HTR when minting new tokens, a transaction fee is charged on each transfer of the newly minted tokens.

# Motivation
[motivation]: #motivation
A user from the Hathor network suggested an alternative to the HTR deposit requirement when minting tokens. The idea is to remove the 1% deposit requirement and, instead, implement a transaction fee for every transfer of the minted tokens.

This change would reduce the upfront cost of minting tokens, making it more accessible to users who may not have sufficient HTR at the time of minting. The network would still benefit from a fee mechanism that contributes to miners’ incentives and overall network security.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## As is

  When a user mints X tokens, they must deposit P% (1%) of X in HTR. If X = 100, then 1 HTR must be deposited. This deposit is returned if the tokens are later melted.

## To be

  Instead of requiring a deposit, minted tokens will be subject to a transaction fee. Every time a transfer occurs with these tokens, a percentage-based fee (to be defined) will be charged. 

## Example:

1.	A user mints 1000 tokens without depositing HTR.
2.	When transferring 500 tokens, a fee (e.g., 0.5% or 1%) is deducted and sent to miners or the network treasury.
3.	This fee structure continues for all transactions involving the newly minted tokens.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
TODO

# Drawbacks
[drawbacks]: #drawbacks

Introducing a transaction fee may discourage frequent transfers of these tokens.

Miners’ incentives might change as fees will be spread over multiple transactions rather than collected upfront.

Tokens minted under different mechanisms will have different behaviors, which could add complexity to user experience and wallet support.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main alternative is to keep the current deposit model. However, this locks up liquidity for token creators, making token issuance less attractive.

Another alternative is a hybrid model, where users can choose between:
	1.	Paying the upfront HTR deposit (current model).
	2.	Opting for the per-transaction fee model (new proposal).

This hybrid approach could offer flexibility while maintaining incentives for both users and miners.

# Prior art
[prior-art]: #prior-art

Ethereum, the most used platform for token issuance, charges a fee to run any operation.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What should the transaction fee percentage be?
- How will fee distribution work?
- How will wallets distinguish between tokens under different models?
- Does make sense having a portion of the fee in a treasury to fund the ecosystem development?
- How will the melt behave?

# Future possibilities
[future-possibilities]: #future-possibilities

TODO

