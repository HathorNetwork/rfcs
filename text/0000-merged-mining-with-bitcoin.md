- Feature Name: merged_mining_with_bitcoin
- Start Date: 2018-12-11
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>, Jan Segre <jan@hathor.network>

# Summary
[summary]: #summary

Enable miners to mine on Hathor while also mining on Bitcoin with no aditional
mining costs.

# Motivation
[motivation]: #motivation

This feature should invite the large Bitcoin mining community and potentially
attract a significant hash power to the Hathor network. Secondly it becomes
generally easier to mine because an unmodified Bitcoin mining client only needs
a merged mining coordinator. Otherwise the client has to be patched, which is
considerably more difficult to do for the variety of clients that exist.

With a higher hash power, we have more security against 51% attacks, and thus
a higher incentive for people to use our network. Which may have some positive
feedback looping.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When mining directly on the Hathor network, a client searchs for a `nonce` that
when appended to `block_without_nonce` can be hashed and the hash is inside a
certain difficulty requirement. The similar is also true on the Bitcoin network,
but the difficulty requirements are different.

Bitcoin mining clients typically accept being told to mine on a different
difficulty than what is specified on the header, because this is useful for
pools to estimate the hash power of a client and divide the rewards accordingly.
Also, on a Bitcoin coinbase, there is space for arbitrary data to be included.

Because of these facts it is possible to include a pointer to Hathor's
`block_without_nonce` on the Bitcoin coinbase such that the result of the work
of a miner can be valid on either network if we accept additional data on the
Hathor block to verify the work originally made for the Bitcoin network.

This mechanism is [commonly known][1] as "merged mining" or "merge mining",
famously implemented on the Namecoin blockchain since 2014. It is typical to say
that Bitcoin is the *parent blockchain* and Namecoin is the *auxiliary
blockchain* in this case. Similarly the individual block used on a merged mining
operation in the parent blockchain is called *parent block* and the set of
additional information needed by the auxiliary blockchain to accept the proof of
work is referred as *AuxPOW* (for auxiliary proof-of-work).

The required changes to the Hathor network is mainly appending optional AuxPOW
data to a block and validating it accordingly. The Hathor network does not need
to interact with the Bitcoin network in any way besides knowing the location of
a few fields and how to build the merkle root of a parent block.

An AuxPOW consists of:
- the Bitcoin coinbase transaction that includes a hash of the Hathor's
  `block_without_nonce`
- the merkle path of that coinbase to rebuild the merkle root
- the bitcoin block header that includes that merkle root

It is necessary to have a daemon that requests mining information to both
networks and builds a valid Bitcoin mining job that clients can work on. This
daemon can be used by solo-miners or mining pool operators. Unfortunately, due
to how mining pools work, it cannot be used by pool members.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Currently the sequence of bytes used to serialize a block, consists of:

| Size     |  Description          | Comments |
|----------|-----------------------|----------|
| variable | `block_without_nonce` | header, graph and script data |
| 16       | `nonce`               | any unsigned 32-bit is a valid value |

The proposed sequence of bytes to serialize a block is:

| Size     |  Description          | Comments |
|----------|-----------------------|----------|
| variable | `block_without_nonce` | unchanged |
| 16       | `nonce`        | `0xff..ff` used to indicate presence of aux\_pow |
| 211+     | `aux_pow`             | optional |

Thus a `nonce` value of `0xff..ff` (2^128 -1) will no longer be accepted. Since
this breaking change will be implemented on a new testnet and there is no
mainnet yet, the only impact is changing the current Hathor mining clients to
skip that value.

The new `aux_pow` structure consists of:

| Size | Description             | Comments |
|------|-------------------------|----------|
| 36   | `bitcoin_header_head`   | first 36 bytes of the header |
| 1+   | `coinbase_tx_head` size | byte length of the next field |
| 47+  | `coinbase_tx_head`      | coinbase bytes before hash of `block_data` |
| 1+   | `coinbase_tx_tail` size | byte length of the next field |
| 18+  | `coinbase_tx_tail`      | coinbase bytes after hash of `block_data` |
| 1+   | `merkle_path` count     | the number of links on the `merkle_path` |
| 32+  | `merkle_path`           | array of links, each one is 32 bytes long |
| 12   | `bitcoin_header_tail`   | last 12 bytes of the header |

## The hashing algorithm
[the-hashing-algorithm]: #the-hashing-algorithm

In order to preserve the property the hash of a block has of representing the
difficulty, we have to change what is hashed. Note that this is different from
what Namecoin does, which is changing how the difficutly is calculated instead
of how the hash is calculated. Also note that the hash function is still the
same, only the function input changes depending if the block has an AuxPOW.

- If there is no AuxPOW there is no change: hash `[block_without_nonce][nonce]`.
- If there is AuxPOW, then hash `[bitcoin_header]`.

## Validation of a block with AuxPOW
[validation-of-a-block-with-auxpow]: #validation-of-a-block-with-auxpow

The only part of `bitcoin_header` that matters is the `merkle_root`. It must
match the merkle root obtained from applying the merkle path to the coinbase
transaction, that is:

```python
calculated_merkle_root = sha256d_hash(coinbase_tx)
for merkle_link in merkle_path:
  calculated_merkle_root = sha256d_hash(merkle_root + merkle_link)
assert calculated_merkle_root == auxpow_merkle_root
```

There is no need to specify the sides of the merkle links because the coinbase
is always the first transaction.

The coinbase transaction must contain the sequence
`[magic_number][block_without_nonce]` in its binary sequence. No specific
location is mandated, but in practice it will usually be on the scriptSig of the
single input. The magic number is defined as the follwing sequence of 4 bytes:
`48 61 74 68` (which is 'Hath' in ASCII).

All of the validations for `block_without_nonce` remain. Notably the difficulty
validation will use the new hash which may not have reached the target
difficulty for the Bitcoin network but must do so for the Hathor network.

## Merged mining coordinator
[merged-mining-coordinator]: #merged-mining-coordinator

We should provide a reference implementation for the coordinator (or proxy) that
Bitcoin mining clients can use without modification.

Our implementation requests information to a Bitcoin fullnode using the Bitcoin
RPC protocol, because the Stratum protocol does not allow a client to
arbitrarily modify the coinbase transaction, which is needed to insert our block
data hash. Information from the Hathor network is retrieved through and
submitted through our custom Stratum protocol, which should be modified
accordingly to allow such operations.

This is outline of how the coordinator works:

- Call [GetBlockTemplate][2] RPC on the Bitcoin fullnode
- Request work to the Hathor stratum server
- Build a coinbase transaction suitable to both, inclusion on the Bitcoin
  network and usable as an AuxPOW:
  - Add `[magic_number][block_without_nonce]` after the [block height][3] on the
    transaction input signature script
- When a stratum client connects, send `mining.set_difficulty` such that work
  can suit the easier network to mine in (most certainly Hathor).
- Send the `mining.notify` identical to what a typical client expects
- When work is submitted check which networks difficulties it is valid for and
  resubmit it to each of them (through [SubmitBlock][4] on Bitcoin and
  `mining.submit` on Hathor)

## Example
[example]: #example

> **TODO**

# Drawbacks
[drawbacks]: #drawbacks

Block size increases by at lease 211 bytes in the case there is an AuxPOW, and
we have one less valid value for `nonce`.  Although this could be reduced by 4
bytes (is it even significant?) when using the version field to indicate the
presence of AuxPOW.

It is somewhat a double-edged sword to make it easier for the mining community
to easily pour hash-power into the Hathor Network, because while we want to
protect the network, larger players can attack the network without having to
dedicate hash power exclusively to do so, that is, while still mining Bitcoin.
But we believe the benefit outweights the risk. Direct partnerships with miners
could also mitigate this risk.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Instead of invalidating a nonce value, we could use the `version` field of the
block to shrink the size of the AuxPOW block. However discussion on the usage of
the `version` field for different purposes is underway, and an RFC may be
proposed soon. Which may prompt a change to this RFC, which should preferrably
be done before accepting it.

- We are using the same specifications of Namecoin and Elastos, what is good
  because people are already used to it.
- Merge mining is good in terms of increasing hash power of the network and also
  good for marketing, since we can argue that you are not using any other energy
  to mine our blocks.

# Prior art
[prior-art]: #prior-art

The merged mining mechanism proposed here is very similar to the one implemented
by several blockchains:

- [Namecoin with Bitcoin][5]
- [Dogecoin with Litecoin][6]
- [Elastos with Bitcoin][7]
- and many others (Rootstock, Devcoin, ixcoin, i0coin, Groupcoin)

Mining pool operators are familiar with the concept, and many do merge mining,
such as:

- [Bitcoin Merged Mining Pool][8]
- [Slush Pool][9]
- [and others][a]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should we change the block mining reward for blocks with AuxPOW? This can be
used to balance the incentive of doing merged mining versus direct mining.

# Future possibilities
[future-possibilities]: #future-possibilities

Doing merged mining with Litecoin is a possibility that is not far in the
future, it mostly depends on accepting more than one hash algorithm.

We may want to support simultaneous merged mining with more than one blockchain.
For example, simultaneously merged mining on Bitcoin Core and Bitcoin Cash.
Althought more work is needed to understand the exact requirements or if it even
is possible.

[1]: https://en.bitcoinwiki.org/wiki/Merged_mining_specification
[2]: https://bitcoin.org/en/developer-reference#getblocktemplate
[3]: https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki
[4]: https://bitcoin.org/en/developer-reference#submitblock
[5]: https://github.com/namecoin/wiki/blob/master/Merged-Mining.mediawiki
[6]: https://www.reddit.com/r/dogecoin/comments/2ci90m/dogecoin_to_enable_auxpow_soon_all_infos_inside/
[7]: https://github.com/elastos/Elastos.ELA/wiki/Merged-mining-guide
[8]: http://mmpool.org/
[9]: https://support.slushpool.com/en-us/section/23-merged-mining
[a]: https://en.bitcoin.it/wiki/Comparison_of_mining_pools
