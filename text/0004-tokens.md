- Feature Name: tokens
- Start Date: 2018-11-07
- RFC PR: [MR !4](https://gitlab.com/HathorNetwork/rfcs/merge_requests/4)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>, Yan Martins <yan.martins@gmail.com>

# Summary
[summary]: #summary

Allow the issuance of other tokens inside Hathor's network, so many tokens would co-exist inside the same DAG of verifications.

# Motivation
[motivation]: #motivation

The motivation is to increase the usability of the network. When anyone may easily issue their own token, they don't need to bother about all details behind maintaining a network. They just focus on their application.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Token transactions will behave as regular transactions in the Hathor DAG. Each new token will have a 32-byte unique identifier (uid). It is possible to set user-friendly alias to these uids, but it is configured in each node/wallet. Users should never rely on alias; the uid of the token must always be checked.

After a token has been created, it may be freely exchanged exactly like Hathor's tokens. Thus, they will have the same features, like Nano Contracts, timelock, and so on. It will be possible to create transactions with more than one token, i.e., Hathor's tokens and a token, as long the verification passes for each token.

Token creation happens in a special transaction output, called initial authority UTXO, giving an address the authority over this token. This authority gives its holder the ability to mint or melt new tokens and may also be transferred to other addresses. As token authority is represented by an UTXO, it can only be used once. This means that using this UTXO as an input to mint tokens, for example, prevents this same output from being used again. If one wishes to keep his authority when minting tokens, he should also create an authority UTXO alongside the other output. Minting a token and removing all authority over it is useful to create tokens with a provable supply limit, as no new tokens can be minted.

This proposal has been largely influenced by Bitcoin Cash's GROUP proposal [1] - they call it a group instead of token.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

To support tokens, the following changes have to be made:
1. In the transaction:  
    a. add a list of token uids. This list is limited to 16 uids;  

2. In the transaction output:  
    a. add field 'token data', which merges 'authority flag' (left-most bit) and 'token uid index' (remaining bits):  
        - authority flag: indicates this is an authority output. Since there's no token transfer in authority outputs, the value field is actually used for token authority flags;  
        - token uid index: references the token uids list in the transaction. We consider that index=0 is always used for hathor, so there's no need to get the token uid (and hathor tokens have no uid);  

Creation of new tokens follows this sequence:
1. Create initial token authority UTXO. As usual, the transaction should have an input (any UTXO). This input is used to compute the token uid (see Token uid section). As the token authority output does not spend the input tokens, there should be another output to send the input amount. During this first step, no new tokens are actually created, only the token uid;
2. The authority UTXO created above is spent to mint new tokens. This is where regular consensus rules are broken and, for the given token, output amount can be greater than input amount. Besides this mint output, other authority output may be included so we still have the ability to mint/melt tokens;
3. After the new tokens have been minted, regular consensus rules apply. This means that, for each token in a transaction, output and input amounts have to match.

## Token uid
The token unique identifier is a 32-byte unique identifier. It's computed in the initial token authority creation, using the sha256 hash of the corresponding input tx id and index. This way, there's no chance of a collision happenning (2 tokens with same token uid). For eg, if the token authority creation output is the second on the transaction output list, we'll use the second input for computing the uid.

## Authority flags

Authority operations break regular consensus rules, in which the sum of the inputs is the same as the outputs. There are 3 authority flags:
1. Create token: create a new token uid;
2. Mint: we're creating new tokens, increasing its supply. Sum of outputs may be greater than inputs;
3. Melt: destroy the tokens. Sum of outputs may be smaller than inputs;

The authority flags reuse the value field in the output, as an authority UTXO does not transfer any tokens. Each bit represents a capability (bit 0 is a least significant bit).

| Bit | Meaning, when enabled (=1) |
|-----|----------------------------|
| 2   | Melt authority             |
| 1   | Mint authority             |
| 0   | Token creation             |

Value field has 4 bytes, but we'll only show the final byte for simplicity:
0b0000: no authority  
0b0001: token creation  
0b0010: mint authority  
0b0100: melt authority  
0b0011: token creation and mint authority  
0b0110: melt and mint authority  

You can only transfer authorities that you have. So an UTXO with just melt authority cannot be used to create a new UTXO with mint authority.

## Examples

In the examples bellow, we will add the token uid alongside the UTXOs so it's easier to follow. In practice, the UTXO only has an index pointing to the token uid in the tx token list.

We'll also consider that we control addresses of the form ADDR_N (ADDR_1, ADDR_2, etc) and have at least one Hathor UTXO, with tx_id and index 0. We'll call sha256(tx_id + 0), the token uid, tuid.

For the output scripts, we use P2PKH (ADDR_N) for the usual pay-to-public-key-hash script with address ADDR_N. This means that whoever wants to spend this UTXO needs to control ADDR_N.

### Creating a limited supply token

We'll first create the authority UTXO, mint 1000000 tokens but do not create any new authority UTXO. This way, it's not possible to mint new tokens.

1. Token authority creation (TX0)
Input: tx_id, 0

| Output | Token uid | Token authority | Amount | Script         |
|--------|-----------|-----------------|--------|----------------|
| 0      | tuid      | 0b0111          | -      | P2PKH (ADDR_1) |
| 1      | 0         | 0               | N      | P2PKH (ADDR_2) |

First output is the creation authority UTXO, giving ADDR_1 authority to mint and melt tokens with tuid. Second output is a regular hathor output, to send the input tokens (considering input had N hathors). This creates the token uid and, being the first output, uses the hash of the first input.

2. Mint operation (TX1)
Input: TX0, 0 (we're spending the first output of the above transaction)

| Output | Token uid | Token authority | Amount       | Script         |
|--------|-----------|-----------------|--------------|----------------|
| 0      | tuid      | 0               | 1000000      | P2PKH (ADDR_3) |

This just mints new tokens and sends it to ADDR_3. This breaks regular consensus rules, but as we're spending an UTXO which has mint authority, that's valid. In this case, all tokens were sent to ADDR_3, but we could have several outputs sending it to different addresses.

It's important to notice that the authority UTXO has been spent and no new authority UTXO was created, so there's no way of minting new tokens.

After the tokens have been minted, they can be exchanged normally, as any hathor is.

3. Distribute token to addresses (TX2)

Input: TX1, 0 (we're spending the first output of the above transaction)

| Output | Token uid | Token authority | Amount       | Script         |
|--------|-----------|-----------------|--------------|----------------|
| 0      | tuid      | 0               | 800000       | P2PKH (ADDR_4) |
| 1      | tuid      | 0               | 200000       | P2PKH (ADDR_5) |

This transaction follows regular consensus rules, ars the sum of inputs has to match sum of outputs.

### Retaining mint authority

1. Token authority creation (TX0)
Input: tx_id, 0

| Output | Token uid | Token authority | Amount | Script         |
|--------|-----------|-----------------|--------|----------------|
| 0      | tuid      | 0b0110          | -      | P2PKH (ADDR_1) |
| 1      | 0         | 0               | N      | P2PKH (ADDR_2) |

Same as last example.

2. Mint operation (TX1)
Input: TX0, 0 (we're spending the first output of the above transaction)

| Output | Token uid | Token authority | Amount       | Script         |
|--------|-----------|-----------------|--------------|----------------|
| 0      | tuid      | 0               | 1000000      | P2PKH (ADDR_3) |
| 1      | tuid      | 0b0110          | -            | P2PKH (ADDR_4) |

The first output is the same as last example, but we're also creating a new authority UTXO (with mint/melt authority) and assigning it to ADDR_4. This means that new tokens might be minted (or melted) by whoever controls ADDR_4.


# Drawbacks
[drawbacks]: #drawbacks

The main drawback is storage usage. Transactions will be bigger, hence they will use more storage.

Another drawback would be an increase in CPU consumption, because the verification algorithm will be more complicated. But it seems to be a minor drawback.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Having the authority and mint/melt operations as regular transactions enables us to use the full power of our network. Some interesting scenarios might be:
1. send the authority UTXO to a multisig address. In case there's a corporation responsible for this token, we can enforce tokens might only be minted/melted when several individuals sign;
2. timelock on token creation. When minting the new tokens, we can create some outputs which are locked and do not create a new authority UTXO. This way the token supply is limited, but it might be released following a schedule;

We can also implement tokens on Hathor without altering the output struct by having an OP_GROUP opcode, as explained in [2] and [3]. This, however, does not have all the authority capabilities, which is a powerful tool for creating and controlling tokens.

# Prior art
[prior-art]: #prior-art

The most widely used platform for creating new tokens is Ethereum, with its ERC-20 [4] and ERC-721 [5] standards. In a way, its implementation is simpler since the smart contract creating the token keeps a balance of all wallet addresses. This is only possible, though, with Ethereum's architecture and couldn't be implemented with Hathor.

On the other side, creating tokens and making operations in Ethereum have costs (called 'gas'). This is big drawback for scalability and adoption.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Sub-tokens, which are non-fungible tokens. We can create groups of tokens which have distinct characteristics within the same token uid;
- Should we have a specific CHILD authority, which enables the 'holder' of such to create new authority outputs? The initial token authority creation gives its holder the power to mint/melt tokens and to distribute this authority. He may wish to give someone else the authority to also mint/melt tokens, but might not want this third-party to also distribute the authority to others.
- The reference Bitcoin Cash document [1] suggests using the output value field for authority flags, instead of having the extra authority field. It's true that only one is used at a time (either value or authority), but I couldn't figure out a way of telling apart when it would be an authority UTXO or token transfer (0b11 could be interpreted as transferring 3 tokens).


# Future possibilities
[future-possibilities]: #future-possibilities

- covenants: I'd like to explore covenants, which are transactions that can enforce restrictions on subsequent transactions. It's briefly explained in [1] and more thoroughly detailed in [6]. Not sure it relates directly to tokens, but might enhance its capabilities.

# References
[references]: #references

[1] https://docs.google.com/document/d/1X-yrqBJNj6oGPku49krZqTMGNNEWnUJBRFjX7fJXvTs/edit#

[2] https://medium.com/@g.andrew.stone/bitcoin-scripting-applications-representative-tokens-ece42de81285

[3] https://www.yours.org/content/colored-coins-in-bitcoin-cash-b26804e05964/

[4] https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md

[5] https://github.com/ethereum/EIPs/issues/721

[6] http://fc16.ifca.ai/bitcoin/papers/MES16.pdf
