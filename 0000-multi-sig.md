- Feature Name: multi-sig
- Start Date: 2018-11-28
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Pedro Henrique Santos Ferreira

# Summary
[summary]: #summary

Enable multi signature transactions and allow them to require M-of-N signatures to be spent.

# Motivation
[motivation]: #motivation

The motivation is that in many cases we need more than one person to be responsible for the funds. With a multi signature transaction people can distribute the responsibility of control and spend the tokens.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

First, to send tokens that can be spend only by multiple signatures, we need to generate a multi sig wallet.

This wallet is generated with N public keys and defining an integer M, that is the minimum quantity of signatures to spent its funds.

With this wallet we define a redeem script, which is the script to be executed to redeem the fund in the wallet and a multi sig address to receive funds.

Multi sig addresses always start with byte 0x05, so we can know if the output will be MultiSig or P2PKH

The sender of the funds does not need to know if it's a multi sig output or not, it will just send to a normal base58 address.

Only when you want to spend the multi sig fund you need to add the minimum number of signatures in the input, so we can validate the script.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The redeem script is defined as `<M> <pubkey 1> <pubkey 2> ... <pubkey N> <N> <OP_CHECKMULTISIG>`

The multi sig address is defined as described in the BIP 13 [https://github.com/bitcoin/bips/blob/master/bip-0013.mediawiki]. The address is `[one-byte version][20-byte hash][4-byte checksum]`, where the [one-byte version] is 0x05 for the multi sig, [20-byte hash] is the hash of the redeem script `RIPEMD160(SHA256(redeemScript))` and the [4-byte checksum] is the first four bytes of the double SHA256 hash of the version and hash. #TODO should we use checksum? I have seen some places that don't use this checksum, will do some tests with bitcoin to check

Multi sig addresses always start with byte 0x05, so we can know if the output will be MultiSig or P2PKH

The output script will be defined as: `OP_HASH160 [20-byte-hash-value] OP_EQUAL`, where the 20-byte-hash-value is the hash of the redeem script, defined by BIP 16 [https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki]

To spend tokens from a multi sig output, the input must have at lease M signatures. The input data must have `<sig1> <sig2> ... <sigM> {redeemScript}`.

To validate that an input can spend an output we first check if it's a multi sig output, verifying if the script matches one multi sig script, then we have two steps to validate the input data:

1. We copy the {redeemScript} and combine with the output script. We will have `{redeemScript} OP_HASH160 [redeemScriptHash] OP_EQUAL`. Executing this, if we have 1 left in the stack, the {redeemScript} is valid and we continue to the next validation;
2. We separate the {redeemScript} in each component and eval the script. We will have `<sig1> <sig2> ... <sigM> <M> <pubkey 1> <pubkey 2> ... <pubkey N> <N> <OP_CHECKMULTISIG>`. 

The OPCODE OP_CHECKMULTISIG will pop the first integer to know how many public keys it expects, then pop all public keys. After that it will pop another integer to know how many signatures it requires, then pop each of the signatures. At last, for each signature, it will search the corresponding public key. If all signatures are valid, the input is valid and it can spend the output fund.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We are using the same specifications of bitcoin, what is good because people are already used to it.
- Multi sig is really necessary when you think about a company or a group that wants to share the responsability of control and spend the funds. Not doing multi sig would limit the use of our tokens.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should we limit the number of maximum public keys accepted in the MultiSig? What about the maximum number of signatures required? The more public keys or signatures we have the bigger the transaction gets
- Bitcoin has some OPCODES for integers (OP_0, OP_1, ..., OP_16). Should we create something like this, or we can begin only pushing an integer (we use more bytes doing that)