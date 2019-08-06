- Feature Name: merged-mining-with-bitcoin
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

When direct minina on the Hathor network, a client searchs for a `nonce` that
when appended to `block_data` can be hashed and the hash is inside a certain
difficulty requirement. The similar is also true on the Bitcoin network, but
the difficulty requirements are different.

Bitcoin mining clients typically accept being told to mine on a different
difficulty than what is specified on the header, because this is useful for
pools to estimate the hash power of a client and divide the rewards accordingly.
Also, on a Bitcoin coinbase, there is space of arbitrary data to be included.o

Because of these facts it is possible to include a pointer to Hathor's
`block_data` on the Bitcoin coinbase such that the result of the work of a miner
can be valid on either network if we accept additional data on the Hathor block
to verify the work originally made for the Bitcoin network.

This mechanism is commonly known as "merged mining" or "merge mining", famously
implemented on [the Namecoin blockchain](https://github.com/namecoin/wiki/blob/master/Merged-Mining.mediawiki)
since 2014. It is typical to say that Bitcoin is the *parent blockchain* and
Namecoin is the *auxiliary blockchain* in this case. Similarly the individual
block used on a merged mining operation in the parent blockchain is called
*parent block* and the set of additional information needed by the auxiliary
blockchain to accept the proof of work is referred as *AuxPOW* (for auxiliary
proof-of-work).

The required changes to the Hathor network is mainly detecting when a block has
AuxPOW and to validate this AuxPOW. The Hathor network does not need to interact
with the Bitcoin network in any way besides knowing the location of a few fields
and how to build the merkle root of a parent block.

An AuxPOW consists of:
- the Bitcoin coinbase transaction that includes a hash of the Hathor's
  `block_data`
- the merkle path of that coinbase to rebuild the merkle root
- the bitcoin block header that includes that merkle root

It is necessary to have a daemon that requests mining information to both
networks and build a valid Bitcoin mining job that clients can work on. This
daemon can be used by solo-miners or mining pool operators. Unfortunately it
cannot be used by pool miners.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Currently the sequence of bytes used to serialize a block, consits of:

| Size     |  Description  | Comments |
|----------|---------------|----------|
| variable | `block_data`  | header, graph and script data |
| 4        | `nonce`       | any unsigned 32-bit is a valid value value |

The proposed sequence of bytes to serialize a block is:

| Size     |  Description  | Comments |
|----------|---------------|----------|
| variable | `block_data`  | unchanged |
| 4        | `nonce`       | `0xffff` is used to indicate presence of aux\_pow |
| 155+     | `aux_pow`     | optional |

Thus a `nonce` value of `0xffff` will no longer be accepted. Since this breaking
change will be implemented on a new testnet and there is no mainnet yet, the
only impact is changing the current Hathor mining clients to skip that value.

The new `aux_pow` structure consists of:

| Size | Description          | Comments |
|------|----------------------|----------|
| 80   | `bitcoin_header`     | validation is slightly different from Bitcoin |
| 1+   | `coinbase_tx` length | byte length of the next field |
| 41+  | `coinbase_tx`        | includes the hash of `block_data` |
| 1+   | `merkle_path` count  | the number of hashes on the `merkle_path` |
| 32+  | `merkle_path`        | each hash is 32 bytes long |

## The hashing algorithm
[the-hashing-algorithm]: #the-hashing-algorithm

If there is no AuxPOW there is no change: hash `[block_data][nonce]`.

If there is AuxPOW, then hash `[bitcoin_header]`.

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

The coinbase transaction must contain the sequence `[magic_number][block_data]`
in its binary sequence. No specific location is mandated, but in practice it
will usually be on the scriptSig of the single input. The magic number is
defined as the follwing sequence of 4 bytes: `48 61 74 68` (which is 'Hath' in
ASCII).

All of the validations for `block_data` remain. Notably the difficulty
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

- Call [GetBlockTemplate] RPC on the Bitcoin fullnode
- Request work to the Hathor stratum server
- Build a coinbase transaction suitable to both, inclusion on the Bitcoin
  network and usable a in AuxPOW:
  - Add `[magic_number][block_data]` after the [block height] on the transaction
    input signature script
- When a stratum client connects, send `mining.set_difficulty` such that work
  can suit the easier network to mine in (most certainly Hathor).
- Send the `mining.notify` identical to what a typical client expects
- When work is submitted check which networks difficulties it is valid for and
  resubmit it to each of them (through [SubmitBlock] on Bitcoin and
  `mining.submit` on Hathor)

# Drawbacks
[drawbacks]: #drawbacks

Block size increases by at lease 155 bytes in the case there is an AuxPOW, and
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

**TODO**

<!--

IDEAS:

- Notorious coins with merged mining support:
  - Namecoin (NMC) with Bitcoin
  - Dogecoin (DOGE) with Litecoin
  - [RootStock (RSK)](https://www.rsk.co/) with Bitcoin
  - to look into:
    - Devcoin (DVC), ixcoin (IXC), i0coin (I0C) and Groupcoin (GRP)
    - UNO, SYS
- Precedent software/pool:
  - [Bitcoin Merge Mining Pool](http://mmpool.org/)
- List comparison of merged mining in practice by large pools

TEMPALTE:

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature
  exist in other programming languages and what experience have their community
  had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us
whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not
on its own motivate an RFC.  Please also take into consideration that rust
sometimes intentionally diverges from common language features.

-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should we change the block mining reward for blocks with AuxPOW? This can be
used to balance the incentive of doing merged mining versus direct mining.

Should we prepare ourselves to allow merged mining as the parent blockchain?
That is, allowing other coins to use our network for merge mining. We would need
a place to add arbitrary data that does not influence when propagating to
the network.

# Future possibilities
[future-possibilities]: #future-possibilities

Doing merged mining with Litecoin is a possibility that is not far in the
future, it mostly depends on accepting more than one hash algorithm.

We may want to support simultaneous merged mining with more than one blockchain.
For example, simultaneour merged mining on Bitcoin Core and Bitcoin Cash.
Althought more work is needed to understand the exact requirements or if it even
is possible.

[GetBlockTemplate]: https://bitcoin.org/en/developer-reference#getblocktemplate
[SubmitBlock]: https://bitcoin.org/en/developer-reference#submitblock
[block height]: https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki

<!-- TODO: sort these references and include them directly on the text  -->
[]: https://www.mycryptopedia.com/coinbase-transaction-explained/
[]: https://en.bitcoin.it/wiki/Merged_mining_specification
[]: https://medium.com/@alexeiZamyatin/a-technical-deep-dive-into-merged-mining-5b67706e1a19
[]: https://bitcoin.stackexchange.com/a/275/19178
[]: https://blog.cyberrepublic.org/2018/12/10/elastos-in-a-nutshell-a-laymans-perspective-merged-mining-part-2-2/
