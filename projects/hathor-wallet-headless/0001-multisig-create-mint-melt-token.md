- Feature Name: MultiSig to assist the EVM compatible bridge by supporting create, mint and melt token commands
- Start Date: 2023-06-14
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Alex Ruzenhack alex@hathor.network

#### Table of content
[table-of-content]: #table-of-content

- [Summary](#summary)
- [Motivation](#motivation)
- [Current system](#current-system)
- [Guide-level explanation](#guide-level-explanation)
- [Drawbacks](#drawbacks)
- [Rationale and alternatives](#rationale-and-alternatives)
- [Prior-art](#prior-art)
- [Unresolved questions](#unresolved-questions)
- [Future possibilities](#future-possibilities)

# Summary
[summary]: #summary

It extends the wallet-headless to support create a custom token, mint and melt token, all using a MultiSig wallet. Add the ability to initialize a MultiSig wallet without seed to support the coordinator service in the operation of sign and push transactions. Also, add an endpoint to inspect transaction data with wallet metadata like balance, etc.

# Motivation
[motivation]: #motivation

It is part of the solution proposed in the [EVM compatible blockchain design](https://github.com/HathorNetwork/rfcs/blob/doc/evm-compatible-brigde/projects/evm-compatible-bridge/design.md#required-features). It aims to enable create, mint and melt tokens using MultiSig wallet and cooperate with the [federation service](https://github.com/HathorNetwork/rfcs/blob/doc/evm-compatible-brigde/projects/evm-compatible-bridge/design.md#required-features) and the coordinator service.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The features we need to add for the MultiSig wallet:
* Create a custom token
* Mint a custom token
* Melt a custom token
* Inspect transaction with wallet metadata
* Initialize wallet without seed

## API endpoints design

Taking by example the `POST:/wallet/p2sh/tx-proposal` and `POST:/wallet/p2sh/tx-proposal/get-my-signatures` endpoints we can identify the following semantic:

- `/wallet` -- the wallet module
- `/p2sh` -- a submodule representing a domain of wallet
- `/tx-proposal` -- a command to create a new transaction proposal
- `/tx-proposal/get-my-signatures` -- this time `/tx-proposal` assumes the role of a component of p2sh domain in the wallet module, and `/get-my-signatures` is the command to sign the transaction with the wallet private key

In summary, we can have either syntax:

* `/module/domain/command`, or
* `/module/domain/component/command`

As our intend is to execute commands, the verb `POST` is the most convenient method to call the endpoints.

## Create custom token

A single wallet can [create a token](https://docs.hathor.network/references/headless-wallet/http-api#operation/createToken) using `POST:/wallet/create-token` endpoint. This operation creates mint and melt authorities by default.

To provide a seamless extension to create a custom token in the MultiSig wallet, the endpoint keeps the same command:
* `POST:/wallet/p2sh/tx-proposal/create-token`

The data fields are:
* `name`*
* `symbol`*
* `amount`*
* `address`
* `change_address`
* `create_mint`
* `mint_authority_address`
* `allow_external_mint_authority_address`
* `create_melt`
* `melt_authority_address`
* `allow_external_melt_authority_address`

*`*` as required field

The response scheme for success:
```ts
{
  "success": boolean,
  "txHex": string
}
```

The purpose of the command is only produce the `txHex` not signed and nothing more, it will not push the transaction to the network. The returned `txHex` will be handled by the coordinator service, and later on by the federation service, which is responsible to sign the transaction.

With this specification we cover the possibility to create custom tokens with or without mint and melt authorities.

## Mint a custom token

A single wallet can [mint a token](https://docs.hathor.network/references/headless-wallet/http-api#operation/mintTokens) using `POST:/wallet/mint-token` endpoint. This operation creates another mint authority by default.

For the MultiSig wallet the endpoint keeps the same command:
`POST:/wallet/p2sh/tx-proposal/mint-token`

The data fields are:
* `token`*
* `amount`*
* `address`
* `change_address`
* `mint_authority_address`
* `allow_external_mint_authority_address`

*`*` as required field

The response scheme for success:
```ts
{
  "success": boolean,
  "txHex": string
}
```

The purpose of the command is only produce the `txHex` not signed and nothing more, it will not push the transaction to the network. The returned `txHex` will be handled by the coordinator service, and later on by the federation service, which is responsible to sign the transaction.

With this specification we cover the possibility to mint tokens, send it to another wallet, create or not another mint authority, and send the authority to another wallet.

## Melt a custom token

A single wallet can melt a token using `POST:/wallet/melt-token` endpoint. This operation creates another melt authority by default.

For the MultiSig wallet the endpoint keeps the same command:
`POST:/wallet/p2sh/tx-proposal/melt-token`

The data fields are:
* `token`*
* `amount`*
* `deposit_address`
* `change_address`
* `melt_authority_address`
* `allow_external_melt_authority_address`

*`*` as required field

The response scheme for success:
```ts
{
  "success": boolean,
  "txHex": string
}
```

The purpose of the command is only produce the `txHex` not signed and nothing more, it will not push the transaction to the network. The returned `txHex` will be handled by the coordinator service, and later on by the federation service, which is responsible to sign the transaction.

With this specification we cover the possibility to melt tokens, send it to another wallet, create or not another melt authority, send the authority to another wallet, and define an address to deposit the withdraw amount if any.

## Decode transaction with wallet metadata

There is an endpoint to [inspect the transaction](https://docs.hathor.network/references/headless-wallet/http-api) by decoding the transaction hexadecimal encode in `POST:/wallet/decode`. However, it doesn't bring signature information, nor wallet information with it.

By extending this endpoint operation to include wallet metadata we introduce a breaking change in the API. However, we keep coherence, once it is used not only to decode a `txHex` but to decode a partial transaction. The extension adds a semantic decoding of the transaction components regarding the wallet. Also, it reduces the roundtrips to the wallet by delivering the tokens balance.

The data field is:
* txHex
* partial_tx

The response scheme for success:
```ts
{
  "success": boolean,
  "completeSignatures": boolean,
  "tx": {
    "version": number,
    "tokens": string[],
    "inputs": [
      {
	    txId: string,
        index: number,
        decoded: {
            type: string,
            address: string,
            timelock: number,
        },  // data from the spent ouput
        token: string,
        value: number,
        tokenData: number, // user face
        token_data: number, // internal use
        script: string,
        signed: boolean,
        mine: boolean,
      },
    ],
    "outputs": [
      {
	    decoded: {
		    address: string,
		    timelock: number,
	    },
        value: number,
        tokenData: number, // user face
        token_data: number, // internal use
        script: string,
        type: string,
        mine: boolean,
        token?: string,
      }
    ],
  },
  "balance": {
    "<token-uid>":  {
      "tokens": { available: number, locked: number ),
      "authorities": {
        mint: { available: number, locked: number ),
        melt: { available: number, locked: number ),
      },
    },
  },
}
```

Lets represent the response document as ` $ `, and each element of a list ` [*] `, we have:
* `$.completeSignatures` -- that represents the completeness of signatures required to use the transaction
* `$.tx.inputs[*].signed` -- that indicates this input has a signature
* `$.tx.inputs[*].mine` -- that indicates this input belongs to this wallet
* `$.tx.outputs[*].mine` -- that indicates this output belongs to this wallet
* `$.balance[*].tokens` -- that contains the balance for the token
* `$.balance[*].authorities` -- that contains the balance of mint and melt for the token

## Initialize read-only MultiSig wallet

This wallet is used by the coordinator service to sign or sign-and-push the transaction given the signatures collected by the service.

A read-only wallet only needs the `xpubkey` to initialize. The MultiSig wallet can be initialized with `seedKey` or `multisigKey`, but initializing with `multisigKey` there is no need to configure the `seeds` property in the configuration.

By combining the two initializations we have a MultiSig read-only wallet, as we can see bellow:
```bash
curl -X POST --data "wallet-id=my1" \
	--data "xpubkey=xpub...FHB" \
	--data "multisigKey=mymultisigwallet" \
	--data "multisig=true" \
	http://localhost:8000/start
```

In the presented configuration the `xpubkey` doesn't have any role in the formation of `multisigData`, it servers only to avoid the requirement to configure the `seeds` for the MultiSig wallet.

With this configuration one can request the creation of transaction proposals but can't call `get-my-signatures` with success. However, after collect all the signatures it is possible to call `sign` or `sign-and-push`.

# Drawbacks
[drawbacks]: #drawbacks

Despite the current design provides a minimal set of operations to enable the EVM compatible bridge, it still lacks some operations like "create authority", "destroy authority", and "delegate authority", that could equip an organization to take full control over its assets in a MultiSig wallet. Also, the token creation don't supports NFT token.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- The addition of specialized commands is better than change the behavior of an existing command because this approach avoids produce breaking changes.

# Prior art
[prior-art]: #prior-art

* This design is a complement for the  [EVM compatible bridge design](https://github.com/HathorNetwork/rfcs/blob/doc/evm-compatible-brigde/projects/evm-compatible-bridge/design.md#federation-service).
* The commands to create token, mint and melt includes a fine tuned authority creation after the PRs [#291](https://github.com/HathorNetwork/hathor-wallet-headless/pull/291) and [#293](https://github.com/HathorNetwork/hathor-wallet-headless/pull/293) in the wallet-headless.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should NFT data be allowed in the create token endpoint for the MultiSig wallet, or do it deserves its own endpoint, as it happens for the single wallet?

# Future possibilities
[future-possibilities]: #future-possibilities

* Destroy authority for a given token
* Delegate authority for a given token
* Create NFT token

## Destroy authority for a given token

Despite there is not similar feature implemented in the single wallet, there is a method to prepare a transaction for this purpose in the wallet facade, which is the `prepareDestroyAuthorityData` method.

For the MultiSig wallet the command endpoint:
`POST:/wallet/p2sh/tx-proposal/destroy-authority`

We can specify the following data fields:
* `type`* -- being it one of `mint|melt` option
* `token`*
* `count`*

*`*` as required field

The response scheme for success:
```ts
{
  "success": boolean,
  "txHex": string
}
```

The purpose of the command is only produce the `txHex` not signed and nothing more, it will not push the transaction to the network. The returned `txHex` will be handled by the coordinator service, and later on by the federation service, which is responsible to sign the transaction.

With this specification we cover the possibility to destroy a melt or mint authority with any count.

## Delegate authority

Despite there is not similar feature implemented in the single wallet, there is a method to prepare a transaction for this purpose in the wallet facade, which is the `prepareDelegateAuthorityData` method.

For the MultiSig wallet the command endpoint:
`POST:/wallet/p2sh/tx-proposal/delegate-authority`

We can specify the following data fields:
* `type`* -- being it one of `mint|melt` option
* `token`*
* `authority_address`*
* `allow_external_authority_address`

*`*` as required field

The response scheme for success:
```ts
{
  "success": boolean,
  "txHex": string
}
```

With this specification we cover the possibility to delegate a melt or mint authority to another wallet.

Another possibility for the delegate action would be the creation of N authorities. The use case for this could be to spread governance power, or increase the transaction throughput for mint or melt. To enable this we can add an optional `count` property to the data fields.

## Create NFT Token

The creation of an NFT token for a MultiSig wallet can happen in either of the paths:
* Extending the create token endpoint to support data field
* Creating an endpoint only for this purpose
