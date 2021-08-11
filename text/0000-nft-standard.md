- Feature Name: nft_standard
- Start Date: 2021-08-02
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

This document presents a standard to be followed when creating an NFT transaction and its metadata. The idea behind having a standard is to be able to identify NFT tokens and to be easy for any plataform that would like to integrate Hathor NFTs.

# Motivation
[motivation]: #motivation

Creating an NFT standard is important mainly for 3 reasons:

1. There are some requirements that an NFT token must fulfill in order to have its digital asset shown in our explorer, so it's important that they are described here.
1. Having a similar structure in most NFTs (also for its metadatas) facilitates the integration with any platform to list and show information about NFTs created on Hathor. 
1. Identifying an NFT is important to show specific information in the explorer and wallets.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Non-fungible tokens (NFTs) are unique, indivisible digital assets created on blockchains. HTR token, for instance, is a fungible token and 1 HTR of mine can be exchanged with 1 HTR of anyone without difference because all tokens are equal. Each NFT’s uniqueness can be proven by a unique identifier.

In Hathor Network an NFT is a new custom token created and its unique identifier is the transaction hash.

## Transaction standard

A transaction can be identified as an NFT creation if it has the following structure:

1. Transaction version is 2 (TOKEN_CREATION_TRANSACTION).
1. All inputs are HTR inputs.
1. First output has HTR value and the script is a pushdata with the NFT data followed by a OP_CHECKSIG.
1. It can optionally have a melt authority output (but never more than one).
1. It can optionally have a mint authority output (but never more than one).
1. It has one or more outputs with the created token (any value is valid).
1. It can optionally have an output for the HTR change.

Any transaction that has this structure will be identified as an NFT and the explorer/wallet screens will show specific NFT information for them. The media associated with the NFT will not be shown unless the NFT goes through Hathor Labs' review process.

## Output script data

The output script data is the most important piece of an NFT transaction because it represents the data that is being uniquely identified in the blockchain.

The data can be any string, e.g. '#A123' as a serial number, or 'https://mywebsite.com/', or even 'ipfs://ipfs/<hash>/filename'.

The most common use case is to represent a digital asset, then the NFT data usually redirects to an image/video/audio file. In that case, in order for us to add the digital asset in our explorer, it's required that this URL uses a immutable protocol, e.g. [IPFS](https://en.wikipedia.org/wiki/InterPlanetary_File_System). **Important to notice that your IPFS URL to be considered immutable must be like ipfs://ipfs/<hash>/filename and not https://ipfs.io/ipfs/<hash>/filename because ipfs.io DNS may change anytime.**

## Metadata

Most NFTs also need extra data besides just the digital asset URL. The metadata standard is important to be followed in order to help future platform integrations with Hathor NFTs. This standard was inspired by [OpenSea metadata standard](https://docs.opensea.io/docs/metadata-standards).

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

### Metadata example

```
{
    "name": "Gandalf",
    "description": "A wizard, one of the Istari order, and the leader and mentor of the Fellowship of the Ring",
    "file": "ipfs://ipfs/QmbuthvFV2EjvfmWXxt2L83PwPPwbjjggBhVsrEB7AXW123/gandalf.png",
    "attributes": [
        {
            "type": "rarity",
            "value": "super rare"
        },
        {
            "type": "hp",
            "value": 32
        },
        {
            "type": "intelligence",
            "value": 99
        },
        {
            "type": "ring",
            "value": 0
        }
    ]
}
```

## Custom NFT

There are some special cases where the NFT token won't follow the proposed standard, e.g. if it needs more than one data output. In that case, our wallets and explorer won't automatically identify this token as NFT.

Given that this situation is expected to be rare, we will handle them manually. The NFT creator will need to get in touch with Hathor team, in order to have the token identified as an NFT on the official Hathor explorer. Besides that, as long as the digital asset's URL is immutable, it should be approved in the review and should be shown in Hathor's Public Explorer like any other standard NFT.