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

Dozer suggested an alternative to the HTR deposit requirement when minting tokens. The idea is to create a new type of custom token where tokens would be minted for free (i.e., no deposits) and fees would be charged for transactions. [RFC](https://github.com/Dozer-Protocol/hathor-rfcs/blob/new-token-economics/projects/new-token-economics/token-economics.md)

This change would reduce the upfront cost of minting tokens, making it more accessible to users who may not have sufficient HTR at the time of minting.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When minting tokens, creators can choose between two transaction models: Deposit-Based and Fee-Based. Each model offers different benefits, depending on how the tokens will be used. To handle this, we will use the same definitions from both [Token Creation Transaction](https://github.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md) and this proposal.

## Deposit-Based Model (as is)

Currently, creators must [deposit a percentage](https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md) (P%) of the minted tokens’ value in HTR. For example, if P% is set at 1% and a user mints 100 tokens, they must deposit 1 HTR. This deposit acts as a reserve and is fully refundable if the tokens are later melted.

## Fee-Based Model

In the fee-based model, no upfront deposit is required — each transfer simply incurs a transaction fee. This gives token creators flexibility to tailor their strategy to specific use cases, such as minting large quantities of memecoins without tying them to HTR value. The model also supports other scenarios that benefit from predictable, low-cost transactions. For example, it enables the creation of scalable in-game currencies or loyalty tokens, where frequent, small transactions don’t incur high fees, and projects can monetize their services more effectively by using custom tokens for fees.

### Fee cost

Fees will be proportional to the number of outputs with fee-based tokens. For instance, if there are 3 HTR outputs, 2 outputs with deposit-based tokens, and another 5 with fee-based tokens, only the latter 5 will count towards the fee.

For melting operations that don't contain any outputs, we'll count them as 1 output in the fee calculation.

This proposal suggests **0.01 HTR per output**.

Apart from accepting HTR for fee payment, any deposit-based token will be accepted. In this case, since the token was created with a 100:1 ratio of HTR ([deposit model](#deposit-based-model-as-is)), the fee needs to be 100x the HTR rate. That means **0.01 HTR or 1.00 deposit-based-token**.

### Fee and melting operations
In the examples below we'll use Fee-based Token (FBT), and Deposit-based Token (DBT) as our tokens.

To accept deposit tokens, the transaction should have an amount >= 100, for example: 
    
    Inputs: [100 FBT, 150 DBT]
    Outputs: [100 FBT, 100 DBT] 
Since 50 DBT represents an withdraw of 0 HTR, this operation should be blocked and considered an attempt to melt without an authority.

For instance, if there is a transaction with:

    Inputs: [1000 FBT, FBT Melt Authority, 500 DBT]
    Outputs: [500 FBT, 400 DBT]

In this scenario, 100 DBT is used to pay the fee without requiring any authority.

For a combination of paying the fee and also melting the token, we'll have the following:

    Inputs: [1000 FBT, FBT Melt Authority, 500 DBT, DBT Melt Authority]
    Outputs: [500 FBT, FBT Melt Authority, 300 DBT, DBT Melt Authority]

Here, 100 DBT is used to pay the fee, and there is a melt of 500 FBT and 100 DBT. For melting tokens, the behavior remains the same, requiring authority.

### Fee destination

The fees will be burned.

## Explorer

Add the fee field in the transaction view, and the token_info_version in the token creation transaction.

## Wallet (desktop, mobile, wallet-lib)

Since Hathor has an easy-to-use approach, we should provide users the option to select between the two models (deposit and fee) when minting. Also, in the wallet, we should always require fee payment in HTR to incentivize HTR demand.

**At the protocol level, other tokens are accepted**.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In this section, technical details are expanded for what was described above.

## Token info version
Given Hathor's [Anatomy of a Transaction RFC](https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md#token-creation), it is reasonable to suggest that the byte used by the token creation transaction `token_info_version` will be used to determine fee-created tokens.

Since each custom token `id` is the hash of the token creation transaction that created it, we can assume the enum values below can be assigned to the `token_info_version` byte in the token creation tx and then we can retrieve it.

So, by adding a TokenInfoVersion enum we have:
- DEPOSIT = 1 (as is)
- FEE = 2

Then, we must allow the versions above in the `deserialize_token_info` method by checking the enum values.

## Transaction token_dict
From the section above, we know how to differentiate between deposit and fee tokens. Based on that, we need to add the input and output data to the current `token_dict` in order to calculate the outputs and check for melting operations without outputs.

## Fee (fee.py)
This new file is responsible for keeping the fee logic, but not for orchestrating it; this will be done in the transaction verifier.

The `should_charge_fee` method will check if the transaction has any fee-based input or output and if the `FEE_FEATURE_FLAG` is enabled.

The `calculate_fee` method will receive a `token_dict` as input, then count the fee-based non-authority outputs and apply a constant value `FEE_OUTPUT_VALUE` to each, returning the total fee in HTR.

It will consider 1 output for melting operations that don't have any output.  
It won't consider mint and melt authorities.

## Transaction verifier 

### verify_sum()
The `verify_sum` method will be modified to match the following based on the `token_dict` result:

- Only calculate a fee value if the `should_charge_fee()` method returns `True`.
- It will call the `calculate_fee()` method from `fee.py` as the `fee`.

For deposit tokens:
- It will sum all the deposit based tokens withdraws with melt authority
- It will sum all the deposit based tokens withdraws without a melt authority:
    -  When the amount of a token is a multiple of 100, otherwise it will be **treated as an attempt of melting without an authority**. 
        - For example, `99 DBT` isn't represents `1 HTR` and an `InputOutputMismatch` error will be raised.
        - Also, `101 DBT` and `199 DBT` represents `1 HTR` and a `InputOutputMismatch` error will be raised, blocking the melting.
- It will **reject** any deposit based token which tries to **mint without an authority**.
- It'll **reject** if the sum of the **withdraws without an authority** are **higher than** the `fee`. If this validation fails, an `InputOutputMismatch` error will be raised.

For fee tokens:
- It will reject any Fee based token which tries to **melt** or **melt** without an authority.

The assertions above are placed to guaranteed the following equality:

    diffHTR = HTR Output - HTR Input
    fee = result of fee calculation
    withdraw_without_authority = sum of withdraw by melting deposit based tokens without an authority
    withdraw = sum of withdraw by melting deposit based tokens with an authority
    deposit = sum of the required HTR to mint deposit based tokens

    diffHTR + fee = withdraw + withdraw_without_authority - deposit


Let's use the same example of the guide level explanation use the following tokens: Deposit Based Token (DBT), and Fee Based Token (FBT).
    
    Inputs: [2 HTR, 100 FBT, 200 DBT, 100 DBT3, 100 DBT3 melt authority]
    Outputs: [
        1 HTR,
        50 FBT, 
        50 FBT,
        200 DBT2,
        DBT2 mint authority,
        DBT2 melt authority
    ]
    Fee: 2 HTR

    diffHTR = 2 HTR
    fee = 2 HTR
    withdraw_without_authority = 2 HTR (resulting from 200 DBT) 
    withdraw = 1 HTR (resulting from 100 DBT3)
    deposit = 2

The balance check is valid: 

    diffHTR + fee = withdraw + withdraw_without_authority - deposit
    -1 + 2 = 1 + 2 - 2
    1 = 1


## Feature flag in settings
For development purposes, this feature will be feature-flagged to run only on the local network by setting the `FEE_FEATURE_FLAG` in settings to true.

## Feature activation
For production, we'll rely on feature activation to release this feature.

## Transaction fee resource
Add an endpoint to calculate fees based on the inputs and outputs, in order to expose the logic used in the transaction verifiers to the wallets.

# Drawbacks

A drawback would be an increase in CPU consumption because the verification algorithm will be more complicated, but it seems to be a minor drawback.

# Prior art

### How Bitcoin deals with fee

Bitcoin handles transaction fees through a system based on competition for limited block space. Each transaction has a weight measured in vBytes, and users set their fees by offering satoshis per vByte, which influences the confirmation speed. During periods of high demand, fees increase due to competition, while in lower traffic periods, they decrease. 

Wallets and services use fee estimators to suggest optimal fees based on the mempool. The protocol does not enforce a minimum fee, but nodes can set a "relay fee" to prevent spam from very low-value transactions.

### How Ethereum deals with fee

Ethereum handles transaction fees through the concept of Gas, where each operation consumes a specific amount of computational resources. Before EIP-1559, users set the Gas Price, and those who paid more had priority. With EIP-1559, a dynamically adjusted Base Fee was introduced based on demand, along with an optional Priority Tip to incentivize validators. 

The Base Fee is burned, reducing the Ether supply, while the Priority Tip goes to validators. This model improves fee predictability, making costs more stable and transparent for users.

# Business unresolved questions
- ~~Should we allow deposit-based tokens to collect fees?~~
- ~~Should we burn or move it to a burn address?~~
- ~~How will the fee be calculated?~~
- ~~Should we only consider the outputs in the fee calculation?~~

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should melt operations be handled for fee-based custom tokens?
- ~~How will fee adjustments be governed?~~
- By removing the transaction fee from mining, how will it affect the tx-mining-service?

# Future possibilities
[future-possibilities]: #future-possibilities

- Custom Fee Markets (letting users optimize fees).

# References

- https://github.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md
- https://github.com/HathorNetwork/rfcs/blob/master/text/0004-tokens.md
- https://github.com/HathorNetwork/rfcs/blob/master/text/0015-anatomy-of-tx.md
- https://bitcoin.org/bitcoin.pdf
- https://eips.ethereum.org/EIPS/eip-1559
- https://ethereum.org/en/developers/docs/gas/
- https://developer.bitcoin.org/devguide/transactions.html#transaction-fees-and-change
