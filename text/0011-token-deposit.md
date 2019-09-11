- Feature Name: token_deposit
- Start Date: 2019-07-04
- RFC PR: https://gitlab.com/HathorNetwork/rfcs/merge_requests/11
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato

# Summary
[summary]: #summary

A transaction that mints `X` tokens consumes `P` percent of `X` in HTR, while a transaction that melts `X` tokens yields `P` percent of `X` in HTR.

As of this writing, `P = 0.01` (i.e., 1%), which means that a deposit of 1 must be made when minting 100 tokens. This deposit will be withdraw if the tokens are melted.

# Motivation
[motivation]: #motivation

To keep Hathor network up and running, we need to give incentives to miners to join our network and mine blocks. Through the deposit, the more users mint new tokens, the higher the demand for native tokens HTR, which is a force to increase the price, which brings more miners to the network.

Instead of charging a fee to create a new token, we demand the token creator to deposit some HTR that can be withdraw if the tokens are melted later. So, when the token creators buy HTR to mint their tokens, they are buying a long term asset, that may be used many times. It is analogous to buying a land and build your company on it. Later on, you can decide to close your company and use your land for another endeavor (or sell it).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

After a token is created, the token creator can perform two operations (among others): mint and melt tokens.

## Mint operation

When minting X tokens, a deposit of p percent of X must be made. This deposit is made by an unbalanced set of inputs and outputs that "disappears with" HTR.

A common transaction that mint tokens has the following structure:

1. Inputs
    1. A mint authority
	2. An input with `Y` HTR
2. Outputs
    1. A mint authority (optional)
	2. An output with `X` tokens being minted
	3. A change output with `Z` HTR, where `Y - Z = ceil(p * X)`

As you can notice, `Z < Y` which means some HTR will be deposited in this transaction.


## Melt operation

When melting X tokens, a withdraw of p percent of X must be made. This withdraw is made by an unbalanced set of inputs and outputs that "creates" HTR.

A common transaction that melt tokens has the following structure:

1. Inputs
    1. A melt authority
	2. An input with X tokens
2. Outputs
    1. A melt authority (optional)
	2. An output with Y tokens, where X - Y are being melted
	3. An output with A HTR, where `A = floor(p * (X - Y))`

As you can notice, `A` HTR will be withdraw from the deposit that was made before.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This feature is implemented in the transaction validation step, i.e., the transaction is valid if, and only if, the deposit/withdraw is correctly made.

One step of the transaction validation is to check the balance of each token in the transaction. Let `balance[T]` be the balance of the token `T`, i.e., the outputs minus the inputs. Then, one out of the three situations are possible: (i) a regular transaction where `balance[T] == 0`, (ii) a minting operation where `balance[T] > 0`, or (iii) a melting operation where `balance[T] < 0`.

To enforce the deposit/withdraw, the following rule is applied:

```
p = 0.01
deposit = sum(ceil(p * balance[T]) for T in tx if T != HTR and balance[T] > 0)
withdraw = sum(floor(p * balance[T]) for T in tx if T != HTR and balance[T] < 0)
valid = (balance[HTR] == withdraw - deposit)
```

The transaction `tx` is valid if, and only if, `valid is True`.

This rule is enough to handle many minting and melting operations in a single transaction `tx`.

We use the `ceil` and `floor` operator to handle the boundary cases where the user is minting/melting an amount `X` such that `p * X < 1` (remember that `X` is always an integer, i.e., X = 1 means 0.01 HTR). So, the `floor` function prevents users from issuing HTR, and the `ceil` function prevents users from depositing zero.

This rounding problem of the deposit, supposedly solved by the `ceil` and `floor` operators, may be used to destroy HTR. For instance, one can create 0.99 tokens which would require a deposit of 0.01 HTR. Then, these tokens are melted yielding zero HTR, therefore destroying 0.01 HTR. We do not think it would be a problem because the attacker would have to have the 0.01 HTR, and they can "destroy their tokens" just destroying the private keys.

# Drawbacks
[drawbacks]: #drawbacks

The drawback of charging a fixed percentage `P` is that not all tokens are worth the same. For example, as of this writing, 1 USD is equal to 108 JPY, which means a stablecoin of JPY would need to mint more tokens than a stablecoin of USD.

Another drawback is that melting authorities can only melt their own tokens, which means they may not be able to withdrawy the whole deposit because they may not have the custody of all tokens that were minted. We've already discussed two alternatives to solve this problem: (i) create a reclaim authority which can be used to reclaim tokens from others, this authority may have other uses besides this problem; or (ii) create a "token destruction transaction", which would withdraw the remaining deposit and prohibit any future transaction involving that token.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative is to charge a fee, which can be done in two ways: (i) melting HTR, or (ii) paying fees to the first block verifying the transaction. We feel that the deposit/withdraw rule is simpler and capable of generating a demand for HTR as the community grows. In this case, when users buy HTR, they are buying the right to mint tokens.

The impact of not doing this is not increasing the demand for HTR as the community grows, which would not increase the price and would not give incentives to miners to join the network.

# Prior art
[prior-art]: #prior-art

As this writing, we do not know any platform using this deposit/withdraw strategy. Ethereum, the most used platform for token issuance, charges a fee to run any operation.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

What should we do when token owners are minting/melting amounts that leads to deposits/withdraws less than 0.01 HTR? The proposed solution is to require a minimum deposit of 0.01 HTR, i.e., round it up when minting, and to withdraw nothing, i.e., round it down when melting. It would allow any one to permanently destroy small amounts of HTR.

An alternative may be to prohibit this cases, but this would require token owners to mint or melt a minimum amount of tokens.

# Future possibilities
[future-possibilities]: #future-possibilities

The percentage P may be adjusted in the future depending on the use cases. Let's say we change P from `P1` to `P2`, and `P2 > P1`. Thus, we need to handle the case in which the deposit was made using `P1`, and a future withdraw using `P2` would yield more HTR than the deposit. To avoid it, we need to know with percentage was used in each case. A possible intuitive idea would be to use use the timestamp of the transactions, but we cannot do it because it can be easily modified. Instead of the timestamp of the transaction, we can safely use the timestamp or the height of the first block that verifies that transaction and would prohibit an old type of transaction.

Another future possibility would be to allow tokens to have a custom precision (number of decimal places). In this case, the deposit and withdraw may be calculated using the smallest unit of the tokens.
