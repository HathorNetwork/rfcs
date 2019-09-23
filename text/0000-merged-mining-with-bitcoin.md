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

Because the Bitcoin block header is so small (80 bytes), there isn't really any
space in the header to add the hash that would tie it to a Hathor block. But
there is space in the coinbase transaction, which is also typically used as
extra nonce on modern mining setups. The Bitcoin block header uses a [merkle
tree][2] to hash the list of transactions included in a block. That means that
we can add a hash the references a Hathor block in a Bitcoin transaction, and
the bundle of transaction with Hathor block hash, merkle path, bitcoin header is
enough to be useful as proof of work for the Hathor block too. That bundle is
the AuxPOW. And the required changes to Hathor is accepting that AuxPOW.

An AuxPOW consists of:
- the Bitcoin coinbase transaction that includes a hash of the Hathor's
  `block_without_nonce`
- the merkle path of that coinbase to rebuild the merkle root
- the bitcoin block header that includes that merkle root

The intended mechanism is havinvg a daemon that requests mining information to
both networks and builds a valid Bitcoin mining job that clients can work on.

This daemon can be used by solo-miners or mining pool operators. Unfortunately,
because of limitations of the stratum protocol (and how mining pools work in
general) it cannot be used by pool members.

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
| variable | `block_without_nonce` | marked with version value of `3` |
| 148+     | `aux_pow`             | optional |

The version value 3 is allocated to indicate a merge mined block. Because a
mining server generates this block, a client has to indicate the intent to merge
mine in order for the server to generate the appropriate block.

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

## Validation of a block with AuxPOW
[validation-of-a-block-with-auxpow]: #validation-of-a-block-with-auxpow

We have to rebuild the Bitcoin block header.

From the `block_without_nonce`, we need the 64 byte sequence used in the
Hathor hashing algorithm:

```python
block_header_without_nonce = sha256_hash(block_funds) + sha256_hash(block_graph)
aux_block_hash = sha256d_hash(block_header_without_nonce)
```

Rebuild the coinbase transaction:

```
coinbase_tx = coinbase_tx_head + aux_block_hash + coinbase_tx_tail
```

The coinbase transaction must contain the sequence
`[magic_number][aux_block_hash]` in its binary sequence. No specific
location is mandated, but in practice it will usually be on the scriptSig of the
single input. The magic number is defined as the follwing sequence of 4 bytes:
`48 61 74 68` (which is 'Hath' in ASCII). That means that the sequence
`48617468` must be at the end of `coinbase_tx_head` and not appear before.

Note: the coinbase transaction will also usually contain an extra nonce used to
increase the hash search space by miners. This extra nonce also uses a section
of the coinbase that does not affect the Bitcoin block (beside the size limit),
the location of the extra nonce is independent from the sequence we need
injected, they can be on any order or even on different parts of the coinbase
(input scriptSig vs an empty output).

Now, calculate the merkle root:

```python
merkle_root = sha256d_hash(coinbase_tx)
for merkle_link in merkle_path:
  merkle_root = sha256d_hash(merkle_root + merkle_link)
```

There is no need to specify the sides of the merkle links because the coinbase
is always the first transaction.

And rebuild the Bitcoin block header:

```
bitcoin_header = bitcoin_header_head + merkle_root + bitcoin_header_tail
```

The bitcoin header hash is used for the difficulty validation of the Hathor
block. All of the validations for `block_without_nonce` remain.

## The hash of a block
[the-hash-of-a-block]: #the-hash-of-a-block

In order to preserve the property the hash of a block has of representing the
difficulty, we have to change what is hashed. Note that this is different from
what Namecoin does, which is changing how the difficutly is calculated instead
of how the hash is calculated. Also note that the hash function is still the
same, only the function input changes depending if the block has an AuxPOW.

- If there is no AuxPOW there is no change: hash
  `[block_header_without_nonce][nonce]`.
- If there is AuxPOW, then hash `[bitcoin_header]`.

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

This is the outline of how the coordinator works:

- Call [GetBlockTemplate][3] RPC on the Bitcoin fullnode
- Request merged mining work to the Hathor stratum server
- Build a coinbase transaction suitable to both, inclusion on the Bitcoin
  network and usable as an AuxPOW:
  - Add `[magic_number][aux_block_hash]` after the [block height][4] on the
    transaction input signature script
- When a stratum client connects, send `mining.set_difficulty` such that work
  can suit the easier network to mine in (most certainly Hathor).
- Send the `mining.notify` identical to what a typical client expects
- When work is submitted check which networks difficulties it is valid for and
  resubmit it to each of them (through [SubmitBlock][5] on Bitcoin and
  `mining.submit` on Hathor)

## Example
[example]: #example

Below is a Hathor block along with an AuxPOW. Notice that the block starts with
the sequence `0003`, which means its version is 3 (merge mined block).

```
+ block_without_nonce:
0003010000032000001976a91433bd9f9291dc2b5b5950cf150190d73292a4087a88ac4035000000
0000005d7a5ff80300000000a99610e50afe968b0aee068d04edabcd14fae83e09ef74319631a779
000250e65d8cb4044a4f5659720179e5faeb13d01476a3fb283c7bb8e57e4d0f0000397625504780
05083fcab58a465b3d83148d067e4c827a96b5eec163540162633636376539646237643265383262
33323661636635376330303830653730662d62346565353136363233393634663937383839613737
636133396565313034622d3864396433376263316563613461616338346537613738363662656630
373432
+ aux_pow:
| bitcoin_header_head:
00000020942cfc6b64f88e2bfa294f6f56951208ea598fd233e738bbb0af8bb600000000
| coinbase_tx_head size:
35
| coinbase_tx_head:
010000000001010000000000000000000000000000000000000000000000000000000000000000ff
ffffff33febb14180048617468
| coinbase_tx_tail size:
35
| coinbase_tx_tail:
00000300000000000000ffffffff0001200000000000000000000000000000000000000000000000
00000000000000000000000000
| merkle_path_count:
0e
| merkle_path:
3cac88bfbf6729594821caaba686cc44c8dc854a842c872e5dcd500120760ef4
856f6c82bc33a1895833e563854c704cffeae513a13a09803fcf2e8b576b20bd
c52d2931e242d3df774369b262668132ff88299b861465ac66fd4dd72476463b
cb1968e6945df3443742489c27d29869cd1c4cd278702d16a55beb444be03a51
c1ed89491808bad0b059fba4bb7124cb37f5201bd78ed478cef656c1d432ffdd
a744fa991d4fe8ab5a22561f9e9ab702ce9532aafb0a2b2d47e636f510afb264
ec3bd60004593be953df8004bd37dd439f687b82fd96205fe992543f62af4efa
0c4aecc8fc521ae1cea4c158998f5a81f3f3693121742f4d3ee976a9c76e6e24
7c11bb417c9454f601c8d2115221b578350133f8b845c2fddcc899f33e1c8272
766b63388c10dab9d348e6ee0e3367f21d38eae44891a089ad172dfc8e85be30
11263db18966a8f0f518e2db16949ca76b9bd6c73637165028283481419269b5
fcb132a6b25a7ca76c0a11e5324fa939eef8e564086a55039ce39120c008cd95
105085c34bb06051c7b6a0b105c91509215a1208c158d5215a6efaa3417aac01
9150168bbef2f5a7911ad7d84fb7dfd07a0cbff4a78b0edcb9d918097e4e481d
| bitcoin_header_tail:
f85f7a5dff5e021a83636db7
```

Rebuilding the bitcoin block header:

```
block_funds:
0003010000032000001976a91433bd9f9291dc2b5b5950cf150190d73292a4087a88ac
block_funds_hash = sha256(block_funds):
6c4163fc6f89fc09f5619b77c4b345fc1f4b10a9f2748daf2b73ff03231c086b
block_graph:
40350000000000005d7a5ff80300000000a99610e50afe968b0aee068d04edabcd14fae83e09ef74
319631a779000250e65d8cb4044a4f5659720179e5faeb13d01476a3fb283c7bb8e57e4d0f000039
762550478005083fcab58a465b3d83148d067e4c827a96b5eec16354016263363637653964623764
326538326233323661636635376330303830653730662d6234656535313636323339363466393738
3839613737636133396565313034622d386439643337626331656361346161633834653761373836
3662656630373432
block_graph_hash = sha256(block_graph):
d85111b39b83615463fc817161556cc5e8b92af8ec425c1f8bfd3e7d21ac6633
base_block_hash = sha256d(block_funds_hash):
23f469c00fb85dc78017a17234b4bcdd81474f6ce96b426e0a77b07f01c1f36a

coinbase_tx = coinbase_tx_head + base_block_hash + coinbase_tx_tail:
010000000001010000000000000000000000000000000000000000000000000000000000000000ff
ffffff33febb1418004861746823f469c00fb85dc78017a17234b4bcdd81474f6ce96b426e0a77b0
7f01c1f36a00000300000000000000ffffffff000120000000000000000000000000000000000000
000000000000000000000000000000000000

coinbase_tx_hash = sha256d(coinbase_tx):
b548ed690f070833fd6b474906898262a04b946ee1f29d4c3824855b9440e2a2

merkle_root = rebuild_merkle_root(coinbase_tx_hash, merkle_path):
| sha256d(a2e240945b8524384c9df2e16e944ba06282890649476bfd3308070f69ed48b5
|         f40e76200150cd5d2e872c844a85dcc844cc86a6abca2148592967bfbf88ac3c)
| = 80a28b92b8d8ba28426b5d0af10a520c5ab204f2812c99a4284063d9c089b29d
| sha256d(9db289c0d9634028a4992c81f204b25a0c520af10a5d6b4228bad8b8928ba280
|         bd206b578b2ecf3f80093aa113e5eaff4c704c8563e5335889a133bc826c6f85)
| = d2ef62af14c87ea7b12bce150594b802de2f1da804956ad75dafe24ba3a711eb
| sha256d(eb11a7a34be2af5dd76a9504a81d2fde02b8940515ce2bb1a77ec814af62efd2
|         3b467624d74dfd66ac6514869b2988ff32816662b2694377dfd342e231292dc5)
| = a45b470110aa16f3f3bac978f0dc13512bec04a3cd82ac43cc3a7bc4d0533c2e
| sha256d(2e3c53d0c47b3acc43ac82cda304ec2b5113dcf078c9baf3f316aa1001475ba4
|         513ae04b44eb5ba5162d7078d24c1ccd6998d2279c48423744f35d94e66819cb)
| = 4448813954042378f8df650b1966d16d4ddb60c64dd6d5daab60caccdf46a00d
| sha256d(0da046dfccca60abdad5d64dc660db4d6dd166190b65dff87823045439814844
|         ddff32d4c156f6ce78d48ed71b20f537cb2471bba4fb59b0d0ba08184989edc1)
| = 571fc922d9967c803f319d863876be926e3210f8970ff9a01acf57fb678b1b55
| sha256d(551b8b67fb57cf1aa0f90f97f810326e92be7638869d313f807c96d922c91f57
|         64b2af10f536e6472d2b0afbaa3295ce02b79a9e1f56225aabe84f1d99fa44a7)
| = 0a44fe08f334f17dbe70ac79018dc1f61bec830019ce73ca7148ccabff4da25b
| sha256d(5ba24dffabcc4871ca73ce190083ec1bf6c18d0179ac70be7df134f308fe440a
|         fa4eaf623f5492e95f2096fd827b689f43dd37bd0480df53e93b590400d63bec)
| = 5447475e7fc1d48b7ec3690926e2c093318dbb256e78eb1f1e29b6af1a095bf8
| sha256d(f85b091aafb6291e1feb786e25bb8d3193c0e2260969c37e8bd4c17f5e474754
|         246e6ec7a976e93e4d2f74213169f3f3815a8f9958c1a4cee11a52fcc8ec4a0c)
| = 3af3914389a58311a77f525c510abf50d4527f981e278a7eec15dd8e0673e2e7
| sha256d(e7e273068edd15ec7e8a271e987f52d450bf0a515c527fa71183a5894391f33a
|         72821c3ef399c8dcfdc245b8f833013578b5215211d2c801f654947c41bb117c)
| = ba14c2eec3f2197cd8b208ec34d5a4df4d54dd41c821c7266826b615c62417f1
| sha256d(f11724c615b6266826c721c841dd544ddfa4d534ec08b2d87c19f2c3eec214ba
|         30be858efc2d17ad89a09148e4ea381df267330eeee648d3b9da108c38636b76)
| = 155fc62bab2377dae090d3517d112087e258c4db17572817fcc5064d0908a2a8
| sha256d(a8a208094d06c5fc17285717dbc458e28720117d51d390e0da7723ab2bc65f15
|         b56992418134282850163736c7d69b6ba79c9416dbe218f5f0a86689b13d2611)
| = d9937177638d60a9dbe406e65436d5ecdf6e9d2cec19b37271347e17b341714e
| sha256d(4e7141b3177e347172b319ec2c9d6edfecd53654e606e4dba9608d63777193d9
|         95cd08c02091e39c03556a0864e5f8ee39a94f32e5110a6ca77c5ab2a632b1fc)
| = e7b194e9c3f4353adb1dcc974fd7cc9041cc108d55af5e5cdad6d16d4993cf83
| sha256d(83cf93496dd1d6da5c5eaf558d10cc4190ccd74f97cc1ddb3a35f4c3e994b1e7
|         01ac7a41a3fa6e5a21d558c108125a210915c905b1a0b6c75160b04bc3855010)
| = 02ccf0f686aecd7192ad1c7d20ec591e83bacfc56fd5a5fe58b76839c7454dc3
| sha256d(c34d45c73968b758fea5d56fc5cfba831e59ec207d1cad9271cdae86f6f0cc02
|         1d484e7e0918d9b9dc0e8ba7f4bf0c7ad0dfb74fd8d71a91a7f5f2be8b165091)
| = efb12445bfe0c2701cd31f5f08dab2874f093e70df3850e5affcdeaa6bac6a1b
1b6aac6baadefcafe55038df703e094f87b2da085f1fd31c70c2e0bf4524b1ef

bitcoin_header = bitcoin_header_head + merkle_root + bitcoin_header_tail:
00000020942cfc6b64f88e2bfa294f6f56951208ea598fd233e738bbb0af8bb6000000001b6aac6b
aadefcafe55038df703e094f87b2da085f1fd31c70c2e0bf4524b1eff85f7a5dff5e021a83636db7
```

Finally, the hash of the Hathor block is the hash of the rebuilt Bitcoin header:

```
sha256d(bitcoin_header):
00000000eeecd88e42e0fd663f23dae809d7bf55c3569ef591583ad1014469b8
```

Validity check:

- Coinbase tx head ends with `Hath`: `...48617468`.
- Hathor block weight is 21, thus the target is `2^(256 - 21) - 1` =
  55213970774324510299478046898216203619608871777363092441300193790394367
  The hash number in base 10 is:
  25161758177604432994098000047175001538137585074666818685206626855352
  Which is less than the target.

# Drawbacks
[drawbacks]: #drawbacks

Block size increases by at least 148 bytes when there is an AuxPOW. Although
this could be reduced by 4 bytes (is it even significant?) when using the
version field to indicate the presence of AuxPOW.

It is somewhat a double-edged sword to make it easier for the mining community
to easily pour hash-power into the Hathor Network, because while we want to
protect the network, larger players can attack the network without having to
dedicate hash power exclusively to do so, that is, while still mining Bitcoin.
But we believe the benefit outweights the risk. Direct partnerships with miners
could also mitigate this risk.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could use the exact same specifications Namecoin and Elastos use. But the
AuxPOW block would be larger, and we're already similar enough that
understanding should not be difficult.

# Prior art
[prior-art]: #prior-art

The merged mining mechanism proposed here is very similar to the one implemented
by several blockchains:

- [Namecoin with Bitcoin][6]
- [Dogecoin with Litecoin][7]
- [Elastos with Bitcoin][8]
- and many others (Rootstock, Devcoin, ixcoin, i0coin, Groupcoin)

Mining pool operators are familiar with the concept, and many do merge mining,
such as:

- [Bitcoin Merged Mining Pool][9]
- [Slush Pool][a]
- [and others][b]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should we change the block mining reward for blocks with AuxPOW? This can be
used to balance the incentive of doing merged mining versus direct mining.

# Future possibilities
[future-possibilities]: #future-possibilities

Doing merged mining with Litecoin is a possibility that is not far in the
future, it mostly depends on how to handle more than one hashing algorithm.

We may want to support simultaneous merged mining with more than one blockchain.
For example, simultaneously merged mining on Bitcoin Core and Bitcoin Cash.
Althought more work is needed to understand the exact requirements or if it even
is possible.

[1]: https://en.bitcoinwiki.org/wiki/Merged_mining_specification
[2]: https://en.wikipedia.org/wiki/Merkle_tree
[3]: https://bitcoin.org/en/developer-reference#getblocktemplate
[4]: https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki
[5]: https://bitcoin.org/en/developer-reference#submitblock
[6]: https://github.com/namecoin/wiki/blob/master/Merged-Mining.mediawiki
[7]: https://www.reddit.com/r/dogecoin/comments/2ci90m/dogecoin_to_enable_auxpow_soon_all_infos_inside/
[8]: https://github.com/elastos/Elastos.ELA/wiki/Merged-mining-guide
[9]: http://mmpool.org/
[a]: https://support.slushpool.com/en-us/section/23-merged-mining
[b]: https://en.bitcoin.it/wiki/Comparison_of_mining_pools
