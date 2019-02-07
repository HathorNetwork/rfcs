- Feature Name: transaction-header
- Start Date: 2019-02-07
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)

# Summary
[summary]: #summary

Hathor's proof of work algorithm hashes the binary string produced by `BaseTransaction`'s `get_struct`.
Bitcoin PoW, for instance, only hashes `80` bytes of data that are called **block header**.

A **transaction header** would be a subset - or function - of transaction 
data that would be the input for all Hathor's proof of work algorithms.

# Motivation
[motivation]: #motivation

Trasaction headers in Hathor would allow miners to calculate the proof of work without the need of:

- Processing the whole binary serialization of the transaction multiple times.
- Knowing all transaction details.

The former point is an optmization that would apply to all miners, multiplying their hashing rate.
But since this enhancement would be applied to all miners, there is no real effect on the network hashing power.

The later point is a real improvemnt, since it allow mining servers to share less transaction information with miner clients.
This approach can have some advantages:

- Fixed data size in mob - allows PoW algorithms to be better optimized, simplified and paralelized.
- No unnecessary transaction information shared with miner.
- Smaller network bandwidth needed to send mining jobs to miners.

## SHA256D

SHA256D is the hash function of Hathor PoW algorithm. It is also the one used on Bitcoin.
An overview and pseudocode of the algorithm can be found on it's [Wikipedia page](https://en.wikipedia.org/wiki/SHA-2#Pseudocode).

### Algorithm Overview

Any binary data with `L` bits (`0 <= L <= 2^64-1`) can be hashed.

Any message should be preprocessed, appending to it one bit with value `1` and `K` bits with value `0`. `K` is the smalles integer such that:

`(L + 1 + K + 64) % 512 == 0`

After the preprocessing the message is split in chunks of `512` bits of data.
The chunks are processed in left to right order, and each chunk alters internal hash state, which impacts on the hash value of following data.

Each chunk is divided in `16` `4`-byte words. Another `48` words are calculated based on the first ones. Each of the `64` words is processed, each altering the internal state. 

### Optimizations

On Bitcoin proof of work, the `nonce` is in the last `4` bytes of data of the block header.
Since the block has `80` bytes of data, the `nonce` is on the second chunk of data.
This means that when miners are increasing the `nonce` and calculating the hash, the first chunk always has the same hash. This gives miners the first optmization:

- Frist chunk state can be precalculated, reducing the number of SHA256 chunks processed from 3 to 2 per SHA256D calculation.

This is called the **midstate**.

There are also `12` bytes - or `3` words - of data before the `nonce` on the second chunk.
Which creates the second - minor - optmization:

- Precalculate the state from the first 3 words of second chunk.

CPU Miner calls it **prehash**.

Both optimizations allow a miner to calculate hash the hash with `65,1%` operations, which gives it a speed up of `53,6%`.

### Conclusion

The position of the `nonce` - and other fields like `timestamp` - and the length of the binary string that is hashed influence the hash rate of the network. 

With a transaction header of fixed size Hathor can avoid any discrepancies among block hash difficulties and facilitate hash calculation optimization.

# Ideas

Some ideas for transaction headers.

## Bitcoin-like

`version, weigth, heigth, input&&output hash, parents hash, timestamp, nonce`

## Minimalist

`SHA256D(tx.get_struct_without_nonce()), nonce`

## Layers

`core_hash = SHA256D(tx.core_stuff())  # Everything but parents (maybe height?), timestamp and nonce`
`tx_header = SHA256D(core_hash + parents_hash + timestamp), nonce`

# Guide-level explanation
`TODO`


# Reference-level explanation
`TODO`


# Rationale and alternatives
`TODO`

# Prior art
`TODO`

# Drawbacks
[drawbacks]: #drawbacks

One may argue that this does not improve the core protocol in any way.
It just adds complexity in order to try to facilitate the miner job.

# Unresolved questions
`TODO`

# Future possibilities
`TODO`