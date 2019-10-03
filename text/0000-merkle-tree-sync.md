- Feature Name: merkle_tree_sync
- Start Date: 2019-10-02
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

An algorithm for syncing the DAG between two full-nodes based on an Augmented Merkle Tree. The primary goal is to reduce the CPU consumption of the sync algorithm, but it will also reduce memory use.

# Motivation
[motivation]: #motivation

Hathor's current implementation of the sync algorithm is based on an Interval Tree. The queries on the Interval Tree apparently are the major CPU consumer, which limits Hathor's network to process 200 tps. For further information, see [Peer-to-peer Protocol: Synchronization](https://gitlab.com/HathorNetwork/rfcs/blob/p2p-sync/text/0000-p2p-sync.md) and [Spike: Synchronization Algorithm is consuming too much CPU](https://gitlab.com/HathorNetwork/rfcs/issues/7).

The expected outcome is a fast algorithm that consumes less CPU than the Interval Tree Sync algorithm.

## Brief analysis of the interval tree sync algorithm

The current sync algorithm uses an exponential search followed by a binary search to find the first timestamp where the DAGs are different.

The exponential search uses `O(log2(dt))` queries, where `dt = now - genesis.timestamp`. It rarely goes all way down and should end in a few queries.

Then, the binary search uses `O(log2(dt/2)` queries. The initial interval of the binary search is gotten from the output of the exponential search. Usually, it is not a long interval.

All that said, the number of queries to find the timestamp is `O(log2(dt)) + O(log2(dt/2)) = O(2 * log2(dt/sqrt(2))) = O(log2(dt))`. Finally, as each query costs `O(log(n) + m)` where `n` is the total number of transactions, and `m` is the number of matches in the search, the overall sync algorithm costs `O(log2(dt) * (log2(n) + m))` per connection. That's the worse case analysis, but usually the search cost will be significantly smaller than `O(log2(dt))`. Notice that `dt` increases by `3600 * 24 * 30 = 2,592,000` per month.

When two nodes are already synced, the algorithm keeps searching for the timestamp. In this case, assuming that no split brain has ocurred, the search takes `O(log(THRESHOLD))` queries, and the overall algorithm takes `O(log2(THRESHOLD) * (log2(n) + m))`. When the tps is high, `m` may be significantly high.

__TODO: Analyze the download algorithm based on the number of queries in the Interval Tree.__


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Description of the tree

The solution is based in a `d`-ary balanced search tree that contains all transactions sorted by `key = (timestamp, hash)`. Let `d` be the number of children of each node of the tree and `n` be the total number of transactions.

Because it is a Merkle Tree, each node of the tree will have the hash of its `d` children. After a new transaction is added, we need to update the hash from the leaf of the transaction to the root, and it takes `O(log_d(n))`. So, the whole add algorithm takes `O(log_d(n))`.

One advantage of using such tree is that it may replace another index that is used to list blocks and transactions in wallets and explorers. Such index is already a tree but cannot be directly used here because it does not have the hashes.

This tree may be implemented using a B+ Tree [1], which is balanced, and can add, search, and remove in `O(log_d(n))`.

## New syncing algorithm

The new syncing algorithm will have two messages: `GET-MERKLE-TREE` and `MERKLE-TREE`.

The `GET-MERKLE-TREE-NODE` message is used to request a node of the tree.

The `MERKLE-TREE-NODE` message will contain the name of the node, the hash of the children's hashes, and a list with the name and hash of each children.

So, the syncing will consists on calling `sync(root)`, where:

```python

from hathor.p2p.downloader import Downloader

class MerkleTreeNode(object):
    hash: bytes  # hash(x.hash for x in children)
    keys: List[NameTuple[timestamp: int, txid: bytes]]  # b keys per node
    children: List[MerkleTreeNode]  # b+1 children per node
    next: Optional[MerkleTreeNode]  # Pointer to the next leaf.
    is_leaf: bool

def sync(remote_node: MerkleTreeNode):
    index: MerkleTreeManager
    downloader: Downloader = manager.downloader

    if remote_node.is_leaf:
        for key in remote_node.keys:
            if not storage.has_transaction(key.txid):
                downloader.get_tx(key.txid)
        return

    local_node: MerkleTreeNode = index.get(remote_node.hash, None)
    if local_node is None or local_node.hash != remote_node.hash:
        for child in node.children:
            sync(child)
```

The downloader must also be able to walk between the sorted transactions, which can be easily done because the leaves are linked in a B+ Tree.


## Sync'ed criteria

Let `T1` and `T2` be two Merkle Tree. Let `min_t(T1, T2)` be the minimum timestamp where `T1` and `T2` are not equal.

Two nodes will be considered synced if `abs(local.latest_timestamp, min_t(local, remote)) < SYNCED_THRESHOLD`.


# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative would be to use the blockchain inside the DAG to sync the full-nodes.

# Prior art
[prior-art]: #prior-art


# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities


# Links
[links]: #links

- [1] https://en.wikipedia.org/wiki/B%2B_tree
