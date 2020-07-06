- Feature Name: merged_mining_with_bitcoin
- Start Date: 2018-12-11
- RFC PR: [HathorNetwork/rfcs#6](https://github.com/HathorNetwork/rfcs/issues/6)
- Hathor Issue: [HathorNetwork/hathor-core#3](https://github.com/HathorNetwork/hathor-core/issues/3)
- Author: Pedro Ferreira <pedro@hathor.network>, Jan Segre <jan@hathor.network>

# Summary
[summary]: #summary

Enable miners to mine both Hathor and Bitcoin at the same time, at the cost of
dedicating a few bytes of the BTC block payload to carry HTR data.

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
we can add a hash that references a Hathor block in a Bitcoin transaction, and
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

Currently the sequence of bytes used to serialize a block consists of:

| Size     |  Description          | Comments |
|----------|-----------------------|----------|
| variable | `block_without_nonce` | header, graph and script data |
| 16       | `nonce`               | any unsigned 32-bit is a valid value |

For merged mining, we have a new block type with `version=3` and the following
structure:

| Size     |  Description          | Comments |
|----------|-----------------------|----------|
| variable | `block_without_nonce` | marked with version value of `3` |
| 148+     | `aux_pow`             | used to reconstruct bitcoin header |

The version value 3 is allocated to indicate a merge mined block. Because a
mining server generates this block, a client has to indicate the intent to merge
mine in order for the server to generate the appropriate block.

The new `aux_pow` structure consists of:

| Size | Description             | Comments |
|------|-------------------------|----------|
| 36   | `bitcoin_header_head`   | first 36 bytes of the header |
| 1+   | `coinbase_tx_head` size | byte length of the next field |
| 47+  | `coinbase_tx_head`      | coinbase bytes before `aux_block_hash` |
| 1+   | `coinbase_tx_tail` size | byte length of the next field |
| 18+  | `coinbase_tx_tail`      | coinbase bytes after `aux_block_hash` |
| 1+   | `merkle_path` count     | the number of links on the `merkle_path` |
| 32+  | `merkle_path`           | array of links, each one is 32 bytes long |
| 12   | `bitcoin_header_tail`   | last 12 bytes of the header |

## Validation of a block with AuxPOW
[validation-of-a-block-with-auxpow]: #validation-of-a-block-with-auxpow

We have to rebuild the Bitcoin block header.

From the `block_without_nonce`, we need the 64 byte sequence used in the
Hathor hashing algorithm:

```python
block_header_without_nonce = sha256(block_funds) + sha256(block_graph)
aux_block_hash = sha256d(block_header_without_nonce)
```

Note: `sha256` above is not a typo.

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
merkle_root = sha256d(coinbase_tx)
for merkle_link in merkle_path:
  merkle_root = sha256d(merkle_root + merkle_link)
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
RPC protocol, because the Stratum protocol does not allow<sup>1</sup> a client
to arbitrarily modify the coinbase transaction, which is needed to insert our
block data hash. Information from the Hathor network is retrieved through and
submitted through our custom Stratum protocol, which should be modified
accordingly to allow such operations.

This is the outline of how the coordinator works:

- Call [GetBlockTemplate][3] RPC on the Bitcoin fullnode
- Request merged mining work to the Hathor stratum server (minor change to our
  current stratum is needed)
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

<sup>1</sup>: The stratum protocol uses an extra nonce to increase the search
space for a fitting hash, because modern hardware go over the nonce field space
too fast.  This extra nonce is simply a sequence of bytes added to the coinbase
transaction in a place that does not affect the outcome of any other
transaction. It is split in two parts, `extra_nonce_1` and `extra_nonce_2`. When
a client subscribes to a server, the server sends the `extra_nonce_1` and the
size of the `extra_nonce_2`, when the client finishes a job, it will send to the
server the `nonce`, `timestamp` (which is also used for extra search space) and
`extra_nonce_2` for a job. Theoretically the `extra_nonce_2` could be used to
insert the data we need to merge mine a block, however there are two major
issues with this: first the size of `extra_nonce_2` is usually 16 bytes and we
would need at the very least 32 bytes (36 as currently defined), also because
more bytes on the coinbase are affected by the size of the extra nonce, it is
not possible for a client to send an `extra_nonce_2` of different size, the
server will always reject it, and second, the `extra_nonce_2` is meant to
increase the search space, if we were to use it to store data, it would defeat
its purpose and mining would become very limited again.

## Example
[example]: #example

Below is an example script (in python3) that parses each field of a merge mined
block and calculates its hash. Notice that the block starts with the sequence
`0003`, which means its version is 3 (merge mined block).

```python
import hashlib, functools, struct, datetime


def sha256(data):
    return hashlib.sha256(data).digest()


def sha256d(data):
    return hashlib.sha256(hashlib.sha256(data).digest()).digest()[::-1]


# example: block 00000000000000001edcb1b01154f8df97199f5fd936d8e7d553d76d01abec3c
# explorer link: https://explorer.hathor.network/transaction/00000000000000001edcb1b01154f8df97199f5fd936d8e7d553d76d01abec3c
# raw_tx: 0003010000190000001976a914bcdddee6f042c73ff4b947bc9bca929fbd4fa98588ac404dea2b257375405ef3846b03000000000000000eab4934332c22f5db57ce7276cc76304384979ff861a5ff5a000000000706cc3cfe08ea41eb25b22d170193a8dfc0335cf3983d97f70ddbe0000000009ff1ea03f91bb20e6b2fc55bcd1f46d0564849641adb6c51f1dc6f4b00000000202f6636c625e1303fffbd016104041dc1e9ca145210940400000000000000000035010000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff33fe05b5090048617468549cf38114100000000000ffffffff0140cd4f260000000016a9c384cd403c314e857d68dc96093c45914c4bbc608701200000000000000000000000000000000000000000000000000000000000000000000000000c78d994a5f520c138408abcb2665aa39df1bbcde26aa13cce9775276371c09d22759c3d2229ea39c7740fbffdf299efaf544f5382099e9dec63b9fbdb83f141b61c00ad5025f5d3ebb15365d42e4acb8654db3a06ace7d638be146c9ba5e5239f64efed174ad8651e05669c15ec258fd31cd1ff79927d2476b59aad5a52cf1f322b70e3bbddb58e474402e2504c5ce13bb408d0c5066186f5796b38522cc0b9ee0965361770473985e6d7ba3417274821245898f13cd3c785a1c3cfbddb0357e48ac9da66991f8b12cbc51fdd47768c930dde106a95672dcfb55adef505f1bdc87c6cec9c7e548007d642495226cb0746e5500dee273bed8c40036bef13f7d6716c221b75f9904d37ed5b55e829ebe23a447847d31d33b645c8787a9eeb9f7195a845b78e5a86ede4c5591c0d741fe889a347dfba8345a6dde9e65427727c65cd0d56f17e358542cfeb3f4702e1d9f0e0e1d24e9f3a632c7b245e6311d96f1a29113709424b0f60dcc6e7e0263815792d13d53b09452db4d672d26c4d2f0dbfe26c84f35ef2d4111798bc21ca

# breakdown:
# - funds:
#   - version:            0003  (merged mining block)
#   - len(outputs):       01
#   - output[0]:
#     - value:            00001900  (64.00 value could be 4 or 8 bytes depending on the amount, check hathor.transaction.base_transaction.bytes_to_output_value)
#     - token_data:       00  (indicates HTR)
#     - len(script):      0019  (0x0019 = 25 bytes)
#     - script:           76a914bcdddee6f042c73ff4b947bc9bca929fbd4fa98588ac  (see hathor.transaction.script for opcodes breakdown)
# - graph:
#   - weight:             404dea2b25737540  (59.82944172036741 struct.unpack('!d', bytes.fromhex('404dea2b25737540'))[0])
#   - timestamp:          5ef3846b  (2020-6-24 13:50:51 datetime.datetime.fromtimestamp(struct.unpack('!I', bytes.fromhex('5ef3846b'))[0])
#   - len(parents):       03  (always 3 for blocks)
#   - parents[0]:         000000000000000eab4934332c22f5db57ce7276cc76304384979ff861a5ff5a  (block)
#   - parents[1]:         000000000706cc3cfe08ea41eb25b22d170193a8dfc0335cf3983d97f70ddbe0  (tx1)
#   - parents[2]:         000000009ff1ea03f91bb20e6b2fc55bcd1f46d0564849641adb6c51f1dc6f4b  (tx2)
#   - len(data):          00
#   - data:               -
# - aux_pow:  (because it is a merged mining block, otherwise it would be a nonce)
#   - header_head:        000000202f6636c625e1303fffbd016104041dc1e9ca1452109404000000000000000000
#   - len(coinbase_head): 35  (varint, means 53, takes 1 bytes, but can take from 1 to 4 bytes)
#   - coinbase_head:      010000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff33fe05b5090048617468
#   - len(coinbase_tail): 54  (varint, means 84, ...)
#   - coinbase_tail:      9cf38114100000000000ffffffff0140cd4f260000000016a9c384cd403c314e857d68dc96093c45914c4bbc60870120000000000000000000000000000000000000000000000000000000000000000000000000
#   - len(merkle_path):   0c  (varint, means 12, which is actually the max length due to bitcoin block size limit)
#   - merkel_path[00]:    78d994a5f520c138408abcb2665aa39df1bbcde26aa13cce9775276371c09d22
#   - merkel_path[01]:    759c3d2229ea39c7740fbffdf299efaf544f5382099e9dec63b9fbdb83f141b6
#   - merkel_path[02]:    1c00ad5025f5d3ebb15365d42e4acb8654db3a06ace7d638be146c9ba5e5239f
#   - merkel_path[03]:    64efed174ad8651e05669c15ec258fd31cd1ff79927d2476b59aad5a52cf1f32
#   - merkel_path[04]:    2b70e3bbddb58e474402e2504c5ce13bb408d0c5066186f5796b38522cc0b9ee
#   - merkel_path[05]:    0965361770473985e6d7ba3417274821245898f13cd3c785a1c3cfbddb0357e4
#   - merkel_path[06]:    8ac9da66991f8b12cbc51fdd47768c930dde106a95672dcfb55adef505f1bdc8
#   - merkel_path[07]:    7c6cec9c7e548007d642495226cb0746e5500dee273bed8c40036bef13f7d671
#   - merkel_path[08]:    6c221b75f9904d37ed5b55e829ebe23a447847d31d33b645c8787a9eeb9f7195
#   - merkel_path[09]:    a845b78e5a86ede4c5591c0d741fe889a347dfba8345a6dde9e65427727c65cd
#   - merkel_path[10]:    0d56f17e358542cfeb3f4702e1d9f0e0e1d24e9f3a632c7b245e6311d96f1a29
#   - merkel_path[11]:    113709424b0f60dcc6e7e0263815792d13d53b09452db4d672d26c4d2f0dbfe2
#   - header_tail:        6c84f35ef2d4111798bc21ca

# calculating the block hash:
#
# this should give an idea of how it works but what is implicit is how to breakdown the raw tx, which will inevitably require parsing of at least the variable size integers to be able to separate funds, graphs and the portions of aux_pow

# base_hash is sha256d(sha256(funds)+sha256(graph)) = 5694f7918e7255dcbde9ba4460ad0a71ab6228f0f6796d2aaf831373f6920277
funds_hash = sha256(bytes.fromhex('0003010000190000001976a914bcdddee6f042c73ff4b947bc9bca929fbd4fa98588ac'))
graph_hash = sha256(bytes.fromhex('404dea2b257375405ef3846b03000000000000000eab4934332c22f5db57ce7276cc76304384979ff861a5ff5a000000000706cc3cfe08ea41eb25b22d170193a8dfc0335cf3983d97f70ddbe0000000009ff1ea03f91bb20e6b2fc55bcd1f46d0564849641adb6c51f1dc6f4b00'))
base_hash  = sha256d(funds_hash + graph_hash)
print('funds_hash   ', funds_hash.hex())
print('graph_hash   ', graph_hash.hex())
print('base_hash    ', base_hash.hex())

# coinbase_hash is sha256d(coinbase_head + base_hash + coinbase_tail)
coinbase = bytes.fromhex('010000000001010000000000000000000000000000000000000000000000000000000000000000ffffffff33fe05b5090048617468') \
           + base_hash \
           + bytes.fromhex('9cf38114100000000000ffffffff0140cd4f260000000016a9c384cd403c314e857d68dc96093c45914c4bbc60870120000000000000000000000000000000000000000000000000000000000000000000000000')
coinbase_hash = sha256d(coinbase)
print('coinbase_hash', coinbase_hash.hex())     # 916af597d66a5e224b9a122436b5f7e2542b5afeee1747566ffb285019b5e1ee

# merkle_root is sha256d applied recursively concatenating the merkle_path
merkle_root = functools.reduce(lambda a, b: sha256d(a[::-1] + b[::-1]), map(bytes.fromhex, [
    '78d994a5f520c138408abcb2665aa39df1bbcde26aa13cce9775276371c09d22',
    '759c3d2229ea39c7740fbffdf299efaf544f5382099e9dec63b9fbdb83f141b6',
    '1c00ad5025f5d3ebb15365d42e4acb8654db3a06ace7d638be146c9ba5e5239f',
    '64efed174ad8651e05669c15ec258fd31cd1ff79927d2476b59aad5a52cf1f32',
    '2b70e3bbddb58e474402e2504c5ce13bb408d0c5066186f5796b38522cc0b9ee',
    '0965361770473985e6d7ba3417274821245898f13cd3c785a1c3cfbddb0357e4',
    '8ac9da66991f8b12cbc51fdd47768c930dde106a95672dcfb55adef505f1bdc8',
    '7c6cec9c7e548007d642495226cb0746e5500dee273bed8c40036bef13f7d671',
    '6c221b75f9904d37ed5b55e829ebe23a447847d31d33b645c8787a9eeb9f7195',
    'a845b78e5a86ede4c5591c0d741fe889a347dfba8345a6dde9e65427727c65cd',
    '0d56f17e358542cfeb3f4702e1d9f0e0e1d24e9f3a632c7b245e6311d96f1a29',
    '113709424b0f60dcc6e7e0263815792d13d53b09452db4d672d26c4d2f0dbfe2',
]), coinbase_hash)[::-1]
print('merkle_root  ', merkle_root.hex())   # b9134cabcd4f8f4b0f8ab442691677d4dc8fcbedc624245dfee446e3405baf99

# block_hash is sha256d(header_head + merkle_root + header_tail)
block_hash = sha256d(
    bytes.fromhex('000000202f6636c625e1303fffbd016104041dc1e9ca1452109404000000000000000000') \
    + merkle_root \
    + bytes.fromhex('6c84f35ef2d4111798bc21ca')
)
print('block_hash   ', block_hash.hex())    # 00000000000000001edcb1b01154f8df97199f5fd936d8e7d553d76d01abec3c
```

Validity check:

- Coinbase tx head contains `Hath` (in this case it's at the very end): `...48617468`.
- Hash is valid. It must be lower than the target (set by the network weight):
```
weight = 59.82944172036741
target = int(2^(256 - 59.82944172036741) - 1) = 113037438696452627388234696612582761940905264201287237369856

block_hash = 00000000000000001edcb1b01154f8df97199f5fd936d8e7d553d76d01abec3c (in hex)
             756736154187959317557784305319446559413951492748219247676 (in base 10)

block_hash < target
```

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
