- Feature Name: nft_standard
- Start Date: 2021-08-02
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

This document presents a standard to be followed when creating a NFT transaction and its metadata. The idea behind having a standard is to be able to identify NFT tokens and to be easy for any plataform that would like to integrate Hathor NFTs.

# Motivation
[motivation]: #motivation

Creating a NFT standard is important mainly for 3 reasons:

1. There are some requirements that a NFT token must fulfill in order to have its digital asset shown in our explorer, so it's important that they are described here.
1. Having a similar structure in most NFTs (also for its metadatas) facilitates the integration with any platform to list and show information about NFTs created on Hathor. 
1. Identifying a NFT is important to show specific information in the explorer and wallets.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Non-fungible tokens (NFTs) are unique, indivisible digital assets created on blockchains. HTR token, for instance, is a fungible token and 1 HTR of mine can be exchanged with 1 HTR of anyone without difference because all tokens are equal. Each NFT’s uniqueness can be proven by a unique identifier.

In Hathor Network a NFT is a new custom token created and its unique identifier is the transaction hash.

## Transaction standard

A transaction can be identified as a NFT creation if it has the following structure:

1. Transaction version is 2 (TOKEN_CREATION_TRANSACTION).
1. All inputs are HTR inputs.
1. First output has HTR value and the script is a pushdata with the NFT data followed by a OP_CHECKSIG.
1. We can have 1 or 0 melt authority output.
1. We can have 1 or 0 mint authority output.
1. We have N outputs with custom token created value (the value can be any number).
1. We can have 1 or 0 HTR output with change value.

Any transaction that has this structure will be identified as a NFT and the explorer/wallet screens will show specific NFT information for them.

## Output script data

The output script data is the most important piece of a NFT transaction because it represents the data that is being uniquely identified in the blockchain.

The data can be any string, e.g. '#A123' as a serial number, or 'https://mywebsite.com/', or even 'ipfs://ipfs/<hash>/filename'.

The most common use case is to represent a digital asset, then the NFT data usually redirects to an image/video/audio file. In that case, in order for us to add the digital asset in our explorer, it's required that this URL uses a immutable protocol, e.g. [IPFS](https://en.wikipedia.org/wiki/InterPlanetary_File_System). **Important to notice that your IPFS URL to be considered immutable must be like ipfs://ipfs/<hash>/filename and not https://ipfs.io/ipfs/<hash>/filename because ipfs.io DNS may change anytime.**

## Metadata

Most NFTs also need extra data besides just the digital asset URL. The metadata standard is important to be followed in order to help future platform integrations with Hathor NFTs.

If the NFT requires a metadata, the output script data saved on blockchain **MUST** be the metadata URL in an immutable protocol (e.g. ipfs as explained in the last section), e.g. ipfs://ipfs/<hash>/metadata.json.

The metadata should have the following JSON structure:

```
{
    "name": {
        "type": "string",
        "description": "Identifies the asset to which this token represents"
    },
    "description": {
        "type": "string",
        "description": "Describes the asset to which this token represents"
    },
    "file": {
        "type": "string",
        "description": "A URL pointing to an immutable resource with a digital asset to which this token represents"
    },
    "attributes": {
        "type": "Array<AttributeObject>",
        "description": "These are the extra attributes for the digital asset. It's an array of AttributeObject to make it as flexible as possible."
    },
}
```

The AttributeObject has the following JSON structure:

```
{
    "type": {
        "type": "string",
        "description": "Type of the attribute, e.g. rarity, stamina, eyes"
    },
    "value": {
        "type": "string | decimal",
        "description": "Value of the attribute, e.g. rare, 1.4, blue"
    }
}
```

The AttributeObject may have more attributes than the ones described above but those are the required ones.

## Custom NFT

There are some special cases where the NFT token won't follow the proposed standard, e.g. if it needs more than one data output. In that case, our wallets and explorer won't automatically identify this token as NFT.

Given that this situation is expected to be rare, we will handle them manually. The NFT creator will need to get in touch with Hathor team on Discord, in order to have the token identified as a NFT. Besides that, if the digital asset URL follows the immutable requirements, it can be shown in the explorer just like any other standard NFT.

# Prior art
[prior-art]: #prior-art

We were using the attributes on metadata JSON as an object but I changed to an array of objects after reading [OpenSea metadata standard](https://docs.opensea.io/docs/metadata-standards) to make it more flexible for future changes.