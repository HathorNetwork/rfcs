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
| variable | `block_without_nonce` | marked with version value of `2` |
| 148+     | `aux_pow`             | optional |

The version value 2 is allocated to indicate a merge mined block. Because a
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
add increase the hash search space by miners. This extra nonce also uses a
section of the coinbase that does not affect the Bitcoin block (beside the size
limit), the location of the extra nonce is independent from the sequence we need
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
the sequence `0002`, which means its version is 2 (merge mined block).

```
+ block_without_nonce:
0002010000032000001976a91433bd9f9291dc2b5b5950cf150190d73292a4087a88ac4035000000
0000005d6fc32303000000005d072a349cad85cf12bbe177ea65a7494b69d768aa0953343abfdda5
000250e65d8cb4044a4f5659720179e5faeb13d01476a3fb283c7bb8e57e4d0f0000397625504780
05083fcab58a465b3d83148d067e4c827a96b5eec163540162633636376539646237643265383262
33323661636635376330303830653730662d30353531366264376338623534656664623262646265
303633663265393566302d3333363336373534323039303436333361653038366438333336393531
643534
+ aux_pow:
| bitcoin_header_head:
00000020143919ea9a20594cea28f5efa11d89faf5f719c3d57ded08e24833e600000000
| coinbase_tx_head size:
35
| coinbase_tx_head:
010000000001010000000000000000000000000000000000000000000000000000000000000000ff
ffffff33fe9f0f180048617468
| coinbase_tx_tail size:
35
| coinbase_tx_tail:
00000100000000000000ffffffff0001200000000000000000000000000000000000000000000000
00000000000000000000000000
| merkle_path_count:
08
| merkle_path:
861b7da94f45e92920cf06285a67bb7a8936bae902a393e57c5735c8a9a43412
95acbb84b83d63cdb98dfbdd0e460511a1e01e929f1ddd60ce1ae39101d2e6cd
24050e27eaa4290bd19156ab9ff8127c0aec49ae656721f6298c8d031159e2d3
343276660615199ec55753c1a72ddd2b52a77de752cd9de19d983906965940bf
34415e61c4c5547b4d596bf71a486bd005518b08ed48578ebc9fa0326d906626
2b7b256f537dff64f9d1364a483de0848b9484d59f65380f4066b27143dea777
a1035d5c89ee34858523cf1859deb4c2043e980493b53660111530abe810dfd7
3b358ae7a8fa32de267431a87d804fdd840ab67a00aea00c84dbaf29cbce7fc0
| bitcoin_header_tail:
23c36f5dff5e021ac23b21d9
```

Rebuilding the bitcoin block header:

```
block_funds:
0002010000032000001976a91433bd9f9291dc2b5b5950cf150190d73292a4087a88ac
block_funds_hash = sha256(block_funds):
9d4fe5fcf9af2f71bdff7b2a0e0fabb86f127cf39025634179ec6d682ef1445d
block_graph:
40350000000000005d6fc32303000000005d072a349cad85cf12bbe177ea65a7494b69d768aa0953
343abfdda5000250e65d8cb4044a4f5659720179e5faeb13d01476a3fb283c7bb8e57e4d0f000039
762550478005083fcab58a465b3d83148d067e4c827a96b5eec16354016263363637653964623764
326538326233323661636635376330303830653730662d3035353136626437633862353465666462
3262646265303633663265393566302d333336333637353432303930343633336165303836643833
3336393531643534
block_graph_hash = sha256(block_graph):
496e6d1abb48731d0356bc22ee91a3da2ce24f44f9d12429502452298a209fa4
base_block_hash = sha256d(block_funds_hash):
6fb3207cd55831fd5831ace92597d675046775b2b57cd34496dde974aa2059fb

coinbase_tx = coinbase_tx_head + base_block_hash + coinbase_tx_tail:
010000000001010000000000000000000000000000000000000000000000000000000000000000ff
ffffff33fe9f0f1800486174686fb3207cd55831fd5831ace92597d675046775b2b57cd34496dde9
74aa2059fb00000100000000000000ffffffff000120000000000000000000000000000000000000
000000000000000000000000000000000000

coinbase_tx_hash = sha256d(coinbase_tx):
9974fd1ab633de61440d5c35f7566a690fec2510b9df767b4bff3648db5771a7

merkle_root = rebuild_merkle_root(coinbase_tx_hash, merkle_path):
| sha256d(a77157db4836ff4b7b76dfb91025ec0f696a56f7355c0d4461de33b61afd7499
|         1234a4a9c835577ce593a302e9ba36897abb675a2806cf2029e9454fa97d1b86)
| = 8b82f3c32200fd5937a1d1e52443837e8f591b158a7685c9211fee831025f522
| sha256d(22f5251083ee1f21c985768a151b598f7e834324e5d1a13759fd0022c3f3828b
|         cde6d20191e31ace60dd1d9f921ee0a11105460eddfb8db9cd633db884bbac95)
| = 866a04a2a9f246cba152c28527ef95527dc4f51b1e5ea083bd2cb6c91b2f4647
| sha256d(47462f1bc9b62cbd83a05e1e1bf5c47d5295ef2785c252a1cb46f2a9a2046a86
|         d3e25911038d8c29f6216765ae49ec0a7c12f89fab5691d10b29a4ea270e0524)
| = f75a1fcc186c703d9db19a68262f87a13706f582bf8d98c8c5b9849cfe3411d5
| sha256d(d51134fe9c84b9c5c8988dbf82f50637a1872f26689ab19d3d706c18cc1f5af7
|         bf4059960639989de19dcd52e77da7522bdd2da7c15357c59e19150666763234)
| = 5e1b3c7a4722587a798eb1ee951368670c5951869f973ca82a0e4bab5972df7e
| sha256d(7edf7259ab4b0e2aa83c979f8651590c67681395eeb18e797a5822477a3c1b5e
|         2666906d32a09fbc8e5748ed088b5105d06b481af76b594d7b54c5c4615e4134)
| = 937da21f2fcd43ce03e8386725fabcd49134dea97658bec8505f27f2e66b68c2
| sha256d(c2686be6f2275f50c8be5876a9de3491d4bcfa256738e803ce43cd2f1fa27d93
|         77a7de4371b266400f38659fd584948b84e03d484a36d1f964ff7d536f257b2b)
| = 3efc5dc815a563b8c380eff19be404d62a726c99565fb218c3770bfe40ef3f5f
| sha256d(5f3fef40fe0b77c318b25f56996c722ad604e49bf1ef80c3b863a515c85dfc3e
|         d7df10e8ab3015116036b59304983e04c2b4de5918cf23858534ee895c5d03a1)
| = ca5966d3ff86e7e3dd9ef47cbed029728addd64f2217d9e87bb95e9daf162141
| sha256d(412116af9d5eb97be8d917224fd6dd8a7229d0be7cf49edde3e786ffd36659ca
|         c07fcecb29afdb840ca0ae007ab60a84dd4f807da8317426de32faa8e78a353b)
| = 9b1f31eacd9f7b3e87e63d51a973314dc87cbc81df4d0227bef91e385d4d5d3e
3e5d4d5d381ef9be27024ddf81bc7cc84d3173a9513de6873e7b9fcdea311f9b

bitcoin_header = bitcoin_header_head + merkle_root + bitcoin_header_tail:
00000020143919ea9a20594cea28f5efa11d89faf5f719c3d57ded08e24833e6000000003e5d4d5d
381ef9be27024ddf81bc7cc84d3173a9513de6873e7b9fcdea311f9b23c36f5dff5e021ac23b21d9
```

Finally, the hash of the Hathor block is the hash of the rebuilt Bitcoin header:

```
sha256d(bitcoin_header):
00000000faf35c4ce3016ed0e37c34a11c405f32e34177f5a9fe3791686fc621
```

Validity check:

- Coinbase tx head ends with `Hath`: `...ffffff33fe9f0f180048617468`.
- Hathor block weight is 21, thus the target is `2**(256 - 21) - 1` =
  5.521397077432451e+70
  The hash number in base 10 is:
  26428185639922524704606880247499117591889977986477406792116037731873 =
  2.6428185639922525e+67
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
