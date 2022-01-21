- Feature Name: tokens
- Start Date: 2018-11-07
- RFC PR: [MR !4](https://gitlab.com/HathorNetwork/rfcs/merge_requests/4)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>, Yan Martins <yan@hathor.network>

# Summary
[summary]: #summary

Allow the issuance of other tokens inside Hathor's network, so many tokens can co-exist inside the same DAG of verifications.

# Motivation
[motivation]: #motivation

Allowing others to create their token on Hathor increases the adoption of the network. When anyone may easily issue their own token, they don't need to bother about all details behind maintaining a network. They just focus on their application.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Token transactions will behave as regular transactions in the Hathor DAG. Each new token will have a 32-byte unique identifier (uid), which is the hash of the transaction that created the token. Besides the uid, the transaction that creates the token will have the name and symbol of this token, so we can always validate this information on the DAG.

After a token has been created, it may be freely exchanged exactly like HTR (Hathor's native token). Thus, they will have the same features, like Nano Contracts, timelock and so on. It will be possible to have transactions with more than one token - e.g. HTR and another token, as long the verification passes for each token.

Tokens have 2 special operations, not allowed for HTR: mint and melt. Minting means creating new tokens, increasing the total supply for this token. Melting, on the other hand, destroys the tokens, decreasing the total supply. Control over these special operations is done using transaction outputs, in this case called authority outputs. You can have an output that allows token mint, melt or both. They are basically the same as regular outputs, but they transfer token authorities and not token amounts. As such, they can only be spent once.

Any attempt to spend an authority output more than once is considered a double-spend attack. This means that using an authority UTXO as an input to mint tokens, for example, prevents this same output from being used again. If one wishes to keep his authority when minting tokens, he should also create an authority output alongside the other output. Minting a token and removing all authority over it is useful to create tokens with a provable supply limit, as no new tokens can be minted.

The token creation happens in a special transaction, whose hash is used as the token uid. This transaction also does the initial mint of tokens. The inputs and outputs are:
1. Inputs:
  . HTR deposit: when minting tokens, we need to deposit some HTR. There may be several inputs to reach the required deposit amount. More about token deposit  [here](https://gitlab.com/HathorNetwork/rfcs/blob/master/text/0011-token-deposit.md);

2. Outputs:
  . Mint: this is the initial supply of the new token;
  . Authorities _(optional)_: output used so the creator can mint/melt tokens in the future. If it's not present, the token will have a fixed supply;
  . Change _(optional)_: if the sum of inputs is larger than the required HTR deposit, there may be one or more outputs for the change.

We can have more than one output of each kind above. For instance, the initial mint amount may be divided across several outputs to send tokens to different addresses. The same applies to the change. Similarly, we can also have several authority outputs. One may contain only the mint flag, another the melt flag or even both combined. We only have to obey the limit of 256 outputs per transaction.

This proposal has been largely influenced by Bitcoin Cash's GROUP proposal [1] - they call it a group instead of token.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

To support tokens, the following changes are made:
1. In the transaction:  
    a. add a list of token uids. This list is limited to 16 uids;  
    b. create a new transaction version for token creation. This will be detailed later;

2. In the transaction output:  
    a. add field 'token_data', which merges 'authority bit' (left-most bit) and 'token uid index' (remaining 7 bits):  
         - authority bit: indicates this is an authority output. Since there's no value transfer in authority outputs, the value field is actually used for token authority flags;  
         - token uid index: references the token's uid in the list of token uids. We consider that index=0 is always used for HTR, so there's no need to get the token uid (and hathor tokens have no uid);  

This last point about HTR always having index=0 creates an off-by-one problem when consulting the tokens uid list. HTR does not have a token uid, so it's never added to the list. Therefore, the first token (other than HTR) will be the first in the list, but its index, on any output, is 1. We always have to account for that when accessing the tokens uid list and use `index - 1`, considering the list starts on index 0.

## Token uid
The token unique identifier is a 32-byte unique identifier. It's defined as the hash of the transaction that created the token. This means we can create only one token per transaction.  

## Authority flags

Authority operations break regular consensus rules, in which the sum of the inputs is the same as the outputs. There are 2 authority flags:
1. Mint: we're creating new tokens, increasing its supply. Sum of outputs may be greater than inputs;
2. Melt: destroy tokens. Sum of outputs may be smaller than inputs;

The authority flags reuse the value field in the output, as an authority UTXO does not transfer any tokens. Each bit represents a capability (bit 0 is a least significant bit).

| Bit | Meaning, when enabled (=1) |
|-----|----------------------------|
| 1   | Melt authority             |
| 0   | Mint authority             |

Output `value` field has 4 bytes, but we'll only show the final byte for simplicity:
0b00000000: no authority  
0b00000001: mint authority  
0b00000010: melt authority  
0b00000011: melt and mint authorities  

You can only transfer authorities that you have. So an UTXO with just melt authority cannot be used to create a new UTXO with mint authority.

## token_data field

On the outputs, the new `token_data` field takes 1 byte and merges the authority bit and token uid index. If the left-most bit is 1, it indicates this is an authority output and the `value` field should be interpreted for authority flags, as described above. A few examples:

0b10000001: an authority output for token with index 1;
0b00000001: a regular output for token with index 1;  
0b00000000: a regular output for token with index 0 (HTR);  
0b10000000: authority output for HTR - _INVALID!_  

## Token creation transaction

Token creation is done in a special transaction, which has some fields and validation different than regular transactions.

| Field             | Size              | Description
| ----------------- | ----------------- | --------------------------
| version           | 2 bytes           | Integer with version
| timestamp         | 4 bytes           | Timestamp in seconds from epoch
| hash              | 32 bytes          | sha256d hash
| nonce             | 4 bytes           | The nonce which solves the PoW correctly
| weight            | 8 bytes           | Float with the transaction weight
| parents           | 64 bytes          | Hashes of the 2 parents
| data              | up to 37 bytes    | For storing token name and symbol
| inputs            | variable          | List of inputs
| outputs           | variable          | List of outputs

Main differences are:
- remove tokens list _(from serialization)_;
- new data field;

We use `version=2` for token creation transactions. The authority outputs may only contain the flags for mint and melt. This prevents this token from being able to use any new operations that are added later on. For example, consider UTXOs with the following flags:

0b00000011: melt and mint authority (valid)  
0b00000111: melt, mint and an undefined authority (INVALID)  

### Tokens list

As mentioned, the tokens list has been removed from serialization, but we still need it when processing the transaction.

It's not possible to have the tokens list on the serialized format because it'd be a circular dependency: we'd need the token uid for calculating the hash, but the new token uid is the hash itself. Therefore, we removed the tokens list from the transaction serialization so it's possible to compute the hash. When processing the transaction, though, we add the tokens list with exactly one token uid, which is equal to the transaction hash.

In summary, these are the steps:
1. Serialize the transaction, without the tokens list;
2. Compute the transaction hash (`tx_hash`);
3. When processing this transaction, consider the tokens list to be `[tx_hash]`

This is important because the new tokens outputs reference the token with index 1 (`token_data` of 0b10000001 or 0b00000001). If we don't have a tokens list on this transaction when processing, we'd get an error of missing token uid.

### Data

This transaction has the `data` field, used for storing additional information about the token. We have a version for this field to indicate what is the expected information in the data. The idea is to allow adding more fields in the future while maintaining backwards compatibility.

On version 1, we store the token name and symbol. Names have between 4 and 30 bytes, while symbols have 2 to 5 bytes. Both can only contain UTF-8 characters, meaning the node will reject any name or symbol which is not a valid UFT-8 string. The format of this field is `<version><name_length><name><symbol_length><symbol>`, where `version`, `name_length` and `symbol_length` are 1-byte unsigned integers.

## Examples

In the examples bellow, we will add the token uid alongside the UTXOs so it's easier to follow. In practice, the UTXO only has an index pointing to the token uid in the tx token list.

We'll also consider that we control addresses of the form ADDR_N (ADDR_1, ADDR_2, etc).

For the output scripts, we use P2PKH (ADDR_N) for the usual pay-to-public-key-hash script with address ADDR_N. This means that whoever wants to spend this UTXO needs to control ADDR_N.

### Creating a limited supply token

We'll create the token and mint 1000 tokens, but do not create any authority UTXO. This way, it's not possible to mint new tokens. We'll consider that we need a 1% HTR deposit, so minting 1000 tokens requires 10 HTR.

1. Token creation tx (TX0)

Input: we need an HTR input with at least 10 HTR. Let's say our input has 15 HTR, so we need 5 HTR as change.

| Output | Token uid | Token authority | Amount | Script         | Description |
|--------|-----------|-----------------|--------|----------------| ------------|
| 0      | TX0.hash  | 0               | 1000   | P2PKH (ADDR_1) | This is the first mint |
| 1      | 0         | 0               | 5      | P2PKH (ADDR_2) | The HTR change output |

This transaction defines the uid of the new token (TX0.hash). We've only showed the outputs on the table above, but TX0 also has the token name and symbol set on the data field.

There are no authority outputs, so no new tokens of TX0.hash can be minted or melted in the future.

2. Distribute token to addresses (TX1)

Input: TX0, 0 (we're spending the first output of the above transaction)

| Output | Token uid | Token authority | Amount    | Script         |
|--------|-----------|-----------------|-----------|----------------|
| 0      | TX0.hash  | 0               | 800       | P2PKH (ADDR_3) |
| 1      | TX0.hash  | 0               | 200       | P2PKH (ADDR_4) |

This transaction follows regular consensus rules, as the sum of inputs has to match sum of outputs.

### Retaining mint authority

1. Token creation tx (TX0)

Input: we need an HTR input with at least 10 HTR. Let's say our input has 10 HTR, so no HTR change is needed.

| Output | Token uid | Token authority | Amount | Script         | Description |
|--------|-----------|-----------------|--------|----------------| ------------|
| 0      | TX0.hash  | 0               | 1000   | P2PKH (ADDR_1) | This is the first mint |
| 1      | TX0.hash  | 0b0011          | -      | P2PKH (ADDR_1) | Output with mint and melt authorities |

2. Mint operation (TX1)

Input: TX0, 1 (we're spending the second output of the above transaction)

| Output | Token uid | Token authority | Amount   | Script         | Description |
|--------|-----------|-----------------|----------|----------------| ----------- |
| 0      | TX0.hash  | 0               | 500      | P2PKH (ADDR_3) | We're minting an additional 500 tokens |
| 1      | TX0.hash  | 0b0011          | -        | P2PKH (ADDR_4) | We retain both mint and melt authorities |

This mints new tokens and sends it to ADDR_3. It breaks regular consensus rules but, as we're spending an UTXO which has mint capability, that's valid. In this case, all tokens were sent to ADDR_3, but we could have several outputs sending it to different addresses.

We also created a new authority UTXO (with mint/melt authority) and assigned it to ADDR_4. This means that new tokens might be minted (or melted) by whoever controls ADDR_4.

After the tokens have been minted, they can be exchanged normally, as any HTR is.


# Drawbacks
[drawbacks]: #drawbacks

The main drawback is storage usage. Transactions will be bigger, hence they will use more storage.

Another drawback would be an increase in CPU consumption, because the verification algorithm will be more complicated, but it seems to be a minor drawback.


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
- Should we enforce only 1 authority output on the token creation tx? Or can we have as many authority outputs as we'd like?
- Should we allow tokens to have authority flags not yet created? Right now we only have mint and melt, but more operations can be created. An authority of 0b0100 is unknown and could be used in the future.


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
