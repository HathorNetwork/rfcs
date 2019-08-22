- Feature Name: Peer-to-Peer Network - Sync
- Start Date: 2019-08-12
- RFC PR: [MR !17](https://gitlab.com/HathorNetwork/rfcs/merge_requests/17)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

Description of the algorithm for two peers to do an initial sync and keep synced in real time.

# Motivation
[motivation]: #motivation

When a new peer connects to the network it must have a way of downloading all past transactions and get in sync with peers that are already connected. Besides that, when a new transaction arrives into the network, it must be propagated through all the peers in the network. This document describes how the peers propagate new transactions and keep their storage in sync with the others.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

During the sync stage both peers must be in the READY state, which is the last one of the connection states, and is responsible to do an initial sync and keep them synced when new transactions (from now on 'transactions' can represent transactions or blocks) arrive in the network.

We assume that a new transaction may arrive at any time and have any timestamp, including old ones.

Two peers are defined to be synced at a timestamp T when they both have the same transactions in all timestamps from the genesis to T. They are defined to be synced if the timestamp of the latest transaction received minus the highest timestamp in which they are synced is smaller or equal than an acceptable threshold

```
latest_timestamp - synced_timestamp <= threshold
```

For example, peers P1 and P2 are synced at timestamp 1564658478 (`synced_timestamp`) and the latest transaction P1 has is from timestamp 1564658492 (`latest_timestamp`), so the difference between their timestamps is 14 seconds which is less than the acceptable `threshold` (currently 60 seconds), so P1 and P2 are synced.

This threshold is important because when we have a high number of transactions per second being exchanged in the network, it's really hard for two peers to have all of them at every moment but that does not mean they are not in sync, they are just updating the latest data. So this threshold is a tolerance to support this situation.

The first step of the sync protocol is to discover the highest timestamp where both peers are synced. To do that, we use an exponential search starting at the current timestamp going backwards, followed by a binary search. We run this algorithm every second because new transactions from the past may arrive at anytime, so we must always check that we are still synced and not just rely on the real time message exchange.

After finding the highest timestamp in which both peers are synced, peers must exchange the missing transactions. It is done in two steps: (i) first, they send each other a list of transactions that will be used to check which transactions are missing, (ii) then, they will download the missing transactions.

The download of the transactions must occur in topological order, which means that every new transaction will be ready to be included in the DAG, i.e., its parents and inputs will all have already be known.

After we are synced, we start receiving and propagating transactions in real time. The sync algorithm keeps running in the background and ensures that the peers are still synced. If, for any reason, the peers get not-synced, the sync algorithm will download the missing transactions until they are synced again.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

During the sync period there are two stages:

1. The syncing phase, where the peer must download all past blocks and transactions from the peer its connecting to, so they both have the same history and get in sync.
2. After the peers are synced, they both receive new transactions in real time when they are propagated into the network.

## Syncing

During the syncing phase the peers can exchange 6 different commands: GET-TIPS, TIPS, GET-NEXT, GET-DATA, DATA.

### GET-TIPS Command

The GET-TIPS command is used to request the tips of a peer in a given timestamp. Its payload is a JSON-formatted string in the following format:

```
{
    "timestamp": 1564658478,
    "include_hashes": false,
    "offset": 0,
}
```

The `timestamp` field is optional and, if not passed, the peer that receives this message will assume it is the latest timestamp.

`include_hashes` is an optional boolean (false by default) that, in case it's true, requests also the hashes of the tips (and not only the `merkle_tree`). These hashes have a pagination limit and that's why we also have the `offset` parameter, so we can request more hashes from the same timestamp.

All GET-TIPS messages expect as a TIPS message back.

### TIPS Command

The TIPS command is used as a reply of a GET-TIPS command and it returns the tips of the requested timestamp. Its payload is a JSON-formatted string in the following format:

```
{
    "length": 12,
    "timestamp": 1564658478,
    "merkle_tree": "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5590",
    "hashes": [],
    "has_more": false,
}
```

The `lenght` is the quantity of tips in the requested `timestamp` and the `merkle_tree` is explained how is calculated below in the 'Find synced timestamp' section.

`hashes` is an array with the hashes of the tips, if requested from the GET-TIPS command, and `has_more` indicates if has more hashes in the requested timestamp (in which case the `offset` parameter should be used to continue requesting in sequence)

When a peer receives a TIPS message, it finds the corresponding GET-TIPS that requested it. Unexpected TIPS messages are discarded.

### GET-NEXT Command

The GET-NEXT command is used to request all the transactions in a given timestamp. Its payload is a JSON-formatted string in the following format:

```
{
    "timestamp": 1564658478,
    "offset": 25,
}
```

This command requests all transactions starting at `timestamp` but may return also some from higher timestamps, in order to make sync faster and avoid sending more messages than necessary.

The reply for this command is paginated, so we don't get back all transactions if there are many in the same timestamp. In this case we use the `offset` field to say to the pagination where to start returning.

For e.g. the first request has the following payload:

```
{
    "timestamp": 1564658478,
    "offset": 0,
}
```

The NEXT message will return 10 hashes (assuming this is the maximum quantity to be returned) but if there are 18 transactions in this timestamp, we will need to keep requesting with a GET-NEXT having the following payload:

```
{
    "timestamp": 1564658478,
    "offset": 10,
}
```

The reply will start in the 10th transaction that has the timestamp `1564658478` and, since we only have 8 transactions left with timestamp `1564658478`, it will also return 2 transactions from the next timestamp (`1564658479`). The next GET-NEXT message will have the following payload:

```
{
    "timestamp": 1564658479,
    "offset": 2,
}
```

All GET-NEXT messages expect a NEXT message back.

### NEXT Command

The NEXT command is used as a reply of a GET-NEXT command and it returns the transactions starting from the requested timestamp and offset. Its payload is a JSON-formatted string in the following format:

```
{
    "timestamp": 1564658478,
    "next_timestamp": 1564658479,
    "next_offset": 2,
    "hashes": [
        "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5592",
        "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5594",
        "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5596",
        ... and 7 more hashes ...
    ],
}
```

- `timestamp`: requested timestamp;
- `next_timestamp`: timestamp to be used as parameter in the next GET-NEXT message. May be different from `timestamp` when hashes from different timestamps are being returned;
- `next_offset`: offset to be used as parameter in the next GET-NEXT message. We may have more than one hash for the same timestamp, so this value says the index of the last hash returned in the `next_timestamp`, so it continues to return the hashes in sequence;
- `hashes`: hashes of the requested transactions.

When a peer receives a NEXT message, it finds the corresponding GET-NEXT that requested it. Unexpected NEXT messages are discarded.

### GET-DATA Command

The GET-DATA command is used to request the data of a transaction. Its payload is a string with the hash of the requested transaction in hexadecimal.

All GET-DATA messages expect a DATA message back.

### DATA Command

The DATA message can be received in two different moments:

- During the sync algorithm and the peer has requested this data and is expecting it;
- During the real time propagation when a new transaction arrives into the network and the peer is not expecting it.

Its payload is a string with the payload type and the transaction struct in hexadecimal in the following format: `payload_type:transaction_struct`, where the`payload_type` is either 'tx' or 'block' to indicate the type of the struct.

### NOT-FOUND Command

The NOT-FOUND command is used when a peer receives a GET-DATA message but does not have the requested transaction. Its payload is a string with the hash of the requested transaction in hexadecimal.

---

There are two steps from the beggining of the syncing until both peers are synced.

1. Find the highest timestamp where both peers are synced, i.e. they have all the transactions of each other from the genesis until this timestamp. This is called `synced_timestamp`
2. Sync the rest of timestamps until now, i.e. download all the transactions where `tx.timestamp > synced_timestamp`

### Find synced timestamp

To find the synced timestamp we start a loop from the latest timestamp and decreasing it until we have a match. We define that two peers are synced in a given timestamp when the merkle tree of their tips are the same.

#### Merkle tree

We calculate the merkle tree as the `sha256` of the hashes of a list of transactions, as described below:

```
merkle = hashlib.sha256()
for h in hashes:
    merkle.update(h)
merkle_tree = merkle.digest()
```

This is not exactly a merkle tree, since we are just calculating the `sha256` of each transaction's hash in sequence but it's good enough to validate the difference between two sets of transactions. We should change this in the future to use a proper merkle tree.

---

To optimize the search, it starts with an exponential search in the timestamp, where in each iteration we decrease the timestamp searched and send a GET-TIPS command to the peer and compare its merkle tree with ours in this timestamp, until we find a timestamp in which both peers are synced (`t0`). In this case, there is an interval `[t0, t1)`, where the peers are synced at `t0` and not-synced at `t1`. Finally, a binary search is used in the `[t0, t1)` interval.

After both searches are finished we will have the highest timestamp where the peers are synced (defined as `synced_timestamp`), then we start downloading the transactions from this timestamp until the latest timestamp.

#### Example

Given two peers (P1 and P2) that are synced at timestamp 1564658493 and P1 latest timestamp is 1564658498 (5 seconds more than the synced timestamp). P1 will try to find in which timestamp they are both synced. The algorithm will execute the following steps:

##### Step 1

Start exponential search

| Searched timestamp | Hashes match? | Decrease | Last timestamp |
| ------------------ | ------------- | -------- | -------------- |
| 1564658498         | No            | 1        | None           |

##### Step 2

| Searched timestamp | Hashes match? | Decrease | Last timestamp |
| ------------------ | ------------- | -------- | -------------- |
| 1564658497         | No            | 2        | 1564658498     |

##### Step 3

| Searched timestamp | Hashes match? | Decrease | Last timestamp |
| ------------------ | ------------- | -------- | -------------- |
| 1564658495         | No            | 4        | 1564658497     |

##### Step 4

| Searched timestamp | Hashes match? | Decrease | Last timestamp |
| ------------------ | ------------- | -------- | -------------- |
| 1564658491         | Yes           | 8        | 1564658495     |

End of exponential search, they are synced at timestamp 1564658491 but are not synced at timestamp 1564658495. So we need to find in the interval `[1564658491, 1564658495)` what's the highest timestamp in which they are synced.

##### Step 5

Start of binary search

| Low timestamp | High timestamp | Searched timestamp | Hashes match? |
| ------------- | -------------- | ------------------ | ------------- |
| 1564658491    | 1564658494     | 1564658493         | Yes           |

So 1564658493 is the highest timestamp in which both peers are synced and the algorithm is stopped here and both peers start syncing from this `synced_timestamp`.

### Sync from synced_timestamp

In this step we iterate until we have downloaded all transactions until the latest timestamp. In each iteration we send a GET-NEXT command to get the hashes of the requested timestamp and offset. After receiving these hashes, we check which one of them the node still don't have and send a request to download them.

This request is made to a download coordinator, who will send a GET-DATA command only one time, replying a deferred that will be resolved when the corresponding DATA command arrives. This prevents from sending a GET-DATA command to all connected peers to transaction.

#### Download coordinator

During the sync stage, when we are downloading unsynced transactions, we might request the same data for more than one connected peer, what ends up flooding the network with repeated and unecessary messages. To solve this problem we've created a download coordinator, which is responsible to receive the requests and avoid sending a duplicate message.

When the coordinator receives a new request, it checkes wether this transaction was already requested and, in case it was, it saves the connection that requested it to return the data after the download is concluded.

A downloaded transaction must be propagated to the network only after all its parents, however a request for a parent data might arrive later and we need to wait for it. To handle this situation, the downloader has two deques to control the order and a sliding window to control the download flow.

The first deque holds the transactions that still need to be downloaded (`waiting_deque`) and the second one the transactions that are being downloaded (`downloading_deque`). The sliding window size is the maximum simultaneous downloads allowed. This proccess is similar to the one used for the [TCP Protocol][1]

##### Example

__Step 1__

Both deques are empty and the following requests arrive: connection C1 requests tx1, tx2 and tx3 and connection C2 requests tx1.

*Waiting deque:* [tx1, tx2, tx3]

*Downloading deque:* []

When the second tx1 request arrives it won't be added again into the waiting_deque, we just save which connection requested to return the data later to it.

__Step 2__

Remove the element from the beginning in the waiting deque and add in the end in the downloading deque, so we keep the same order.

*Waiting deque:* [tx2, tx3]

*Downloading deque:* [tx1]

We start downloading tx1.

__Step 3__

The tx1 download finishes and, since it's the first element of the downloading_deque, we don't need to wait for any other download to finish. So we remove it from the deque, propagate to the network and return it to C1 and C2. Then we remove tx2 from the waiting_deque and add to the downloading_deque.

*Waiting deque:* [tx3]

*Downloading deque:* [tx2]

We start downloading tx2.

__Step 4__

The window size is bigger than 1, so we can start another download while we are still waiting for tx2 to finish.

*Waiting deque:* []

*Downloading deque:* [tx2, tx3]

We start downloading tx3.

__Step 5__

The tx3 download finishes but it's not the first element in the downloading_queue, so we need to wait for tx2 to finish before propagating tx3 to the network.

__Step 6__

Download of tx2 finishes, so we propagate tx2 and tx3 to the network and return both to C1.

##### Error handling

When sending a request for data to a peer we might have some errors. The peer might not have the requested transaction (we receive back a NOT-FOUND message) or the request could also takes more time than the timeout. In this case, when we have more than one connection waiting for a transaction that had an error, we start a new download for the same transaction to a different peer.

---

After theses two steps are completed, both peers are in sync. Every second the two steps above are executed again to validate that they are still synced. This check that two peers are synced is made with the following formula:

```
tx_storage.latest_timestamp - synced_timestamp <= sync_threshold
```

If this is true, the peers are synced. Where `tx_storage.latest_timestamp` is the highest timestamp of a transaction that your node has in the storage and the `sync_threshold` is a delta value of acceptance that they might be unsync. Right now this value is 60 seconds, i.e. two synced peers can have a difference of transactions and blocks at most of one minute.

## Real Time

New transactions may arrive at any time in the network and, when they were not requested by the peer, it means that is a new transaction that must be propagated to the connected peers. We only propagate the transactions to a peer in some conditions:

- You are in sync with this peer;
- You are not synced but this peer is in the READY state and your `synced_timestamp` is higher than the parents timestamps, i.e. the destination peer must have all the parents before downloading the new transaction.

The propagation is done sending a DATA message to the destination peer with the new transaction. It follows a priority order explained below.

### Propagation Priority

Each peer connection has two different queues, one with higher priority over the other. The one with priority is used to add blocks and their parents (in case they are not downloaded yet), while transactions are added to the lower priority one. When we are propagating a transaction we first check if there is any element in the priority queue and send all of them and, only after this queue is empty, we start sending the elements from the other.

#### Example

Given peer P1 that already has block B1 and transactions Tx1, Tx2 and Tx3. Some new blocks and transactions arrive from the network in the following order:

| Transaction | Parents      |
| ----------- | ------------ |
| B2          | B1, Tx1, Tx2 |
| Tx4         | Tx2, Tx3     |
| B3          | B2, Tx4, Tx6 |
| Tx5         | Tx3, Tx4     |
| B4          | B3, Tx5, Tx7 |
| Tx6         | Tx4, Tx5     |
| Tx7         | Tx4, Tx6     |

1. B2 arrives and is added to the priority queue, because it's a block;
2. Tx4 arrives and is added to the normal queue, because it's a transaction;
3. B3 arrives and will be added to the priority queue (because it's a block) but Tx5 is one of its parents and the peer must download it before downloading B3. So we will also add Tx5 to the priority queue before B3 (even though it's a transaction);
4. B4 arrives and will be added to the priority queue but Tx7 is one of its parents and the peer must download it before B4. However, Tx7 also has a parent (Tx6) that is not downloaded yet by the peer. So we first add Tx6 to the priority queue, then Tx7 and finally B4.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- We are currently accepting transaction from any time in the past. How should we define a threshold to reject past transactions while still being able to recover from a split brain that lasted more than this threshold.

# Drawbacks
[drawbacks]: #drawbacks

- Currently we are connecting to all available peers in the network and each peer propagates a new transaction to all of its connections, so it requires O(n^2) propagations of each new transaction, where n is the number of peers connected.

# Future possibilities
[future-possibilities]: #future-possibilities

- We should not connect to all available peers, or we should not propagate the new transaction to all connected peers in the real time sync. This floods the network with repeated messages.
- GET-TIPS command could accept more than one timestamp as payload, to reduce the number of messages exchanged and get tips from many timestamps faster.
- We should cache the interval tree, which is responsible for storing the tips in a given timestamp. The biggest problem of that is to invalidate this cache when an old transaction arrives.
- NEXT command could return also the timestamp of each hash and, in the downloader, we can know what is the timestamp of each hash, so when we have a situation like `tx1` and `tx2` with the same timestamp `t` and the downloading deque is `[tx1, tx2]`. If the `tx2` download finishes first, we can safely add it to the DAG before `tx1`, since they have the same timestamp.
- Nowadays, when two peers (P1 and P2) start a connection, the syncing phase is done by both of them. The stages find synced timestamp and sync from timestamp is done twice but if P1 is synced with P2 at timestamp T, P2 will be synced with P1 at timestamp T also. So we can do this phase only in one peer and then send a message to the other indicating the result.

[1]: https://en.wikipedia.org/wiki/Sliding_window_protocol
