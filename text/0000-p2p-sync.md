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

When a new peer connects to the network it must have a way of download all past transactions and get in sync with peers that are already connected. Besides that, when a new transaction arrives into the network, it must be propagated through all the peers in the network. This document describes how the peers propagate new transactions and keep their storage in sync with the others.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

During the sync stage both peers must be in the READY state, which is the last one of the connection states, and is responsible to do an initial sync and keep them synced when new transactions (from now on 'transactions' can represent transactions or blocks) arrive in the network.

Two peers are defined to be synced in a timestamp T when they both have the same transactions in all timestamps from the genesis to T. They are defined to be synced if the highest timestamp when they are synced is bigger than the current timestamp minus a delta value.

The first step of the sync protocol is to discover until what timestamp both peers are already synced between them. To discover that we iterate from the current timestamp until the genesis to see the first timestamp when the tips of both peers match. We run this algorithm every second because new transactions can arrive in the past, so we must always check that we are synced and not just rely on the real time message exchange.

After discover the first timestamp both peers are synced, we must download all the transactions from the unsynced timestamps. We request a list of transaction passing the synced timestamp and it returns a list of hashes. We validate which of these hashes I need to download and request the full data for them. We execute this algorithm until the peers are synced.

From this first moment when we are synced, we keep receiving all transactions the are propagated into the network in real time and check every second that both peers are still in sync. This is important for the case that some problem happened when propagating a transaction and one of the peers don't receive the message. In the real time syncing blocks have priority to be propagated and sent to peers, so full nodes can mine with the newest data.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

During the sync period there are two stages:

1. The syncing phase, where the peer must download all past blocks and transactions from the peer its connecting to, so they both have the same history and get in sync.
2. After the peers are synced, they both receive new transactions in real time when they are propagated into the network.

## Syncing

During the syncing phase the peers can exchange 6 different commands: GET_TIPS, TIPS, GET_NEXT, GET_DATA, DATA.

### GET_TIPS Command

The GET_TIPS command is used to request the tips of a peer in a given timestamp. Its payload is a JSON-formatted string in the following format:

```
{
    "timestamp": 1564658478,
}
```

The `timestamp` field is optional and, if not passed, the peer that receives this message will assume is the latest timestamp.

All GET_TIPS messages are stored in a deferred dictionary and return a deferred that resolves when the expected reply (a TIPS message) arrives.

### TIPS Command

The TIPS command is used as a reply of a GET_TIPS command and it returns the tips of the requested timestamp. Its payload is a JSON-formatted string in the following format:

```
{
    "length": 12,
    "timestamp": 1564658478,
    "merkle_tree": "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5590",
}
```

The `lenght` is the quantity of tips in the requested `timestamp` and the `merkle_tree` is explained how is calculated below in the 'Find synced timestamp' section.

When a peer receives a TIPS message, it searches for a deferred of a GET_TIPS to be resolved.

### GET_NEXT Command

The GET_NEXT command is used to request all the transactions in a given timestamp. Its payload is a JSON-formatted string in the following format:

```
{
    "timestamp": 1564658478,
    "offset": 25,
}
```

The reply for this command is paginated, so we don't get back all transactions if there are many in the same timestamp. In this case we use the `offset` field to say to the pagination where to start returning.

For e.g. the first request has the following payload:

```
{
    "timestamp": 1564658478,
    "offset": 0,
}
```

The NEXT message will return 10 hashes (assuming this is the maximum quantity to be returned) but if there are 22 transactions in this timestamp, we will need to keep requesting with a GET_NEXT having the following payload:

```
{
    "timestamp": 1564658478,
    "offset": 10,
}
```

The reply will start in the 10th transaction that has the timestamp `1564658478`.


All GET_NEXT messages are stored in a deferred dictionary and return a deferred that resolves when the expected reply (a NEXT message) arrives.

### NEXT Command

The NEXT command is used as a reply of a GET_NEXT command and it returns the transactions starting from the requested timestamp and offset. Its payload is a JSON-formatted string in the following format:

```
{
    "timestamp": 1564658478,
    "next_timestamp": 1564658479,
    "next_offset": 12,
    "hashes": [
        "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5592",
        "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5594",
        "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5596",
    ],
}
```

The `next_timestamp` and `next_offset` are the parameters to be used in the following GET_NEXT message, so it continues getting the hashes in sequence. The `hashes` are the hashes of the requested transactions.

When a peer receives a NEXT message, it searches for a deferred of a GET_NEXT to be resolved.

### GET_DATA Command

The GET_DATA command is used to request the data of a transaction. Its payload is a string with the hash of the requested transaction in hexadecimal.

All GET_DATA messages are stored in a deferred dictionary (each key is unique for each transaction request) and return a deferred that resolves when the expected reply (a DATA message) arrives.

### DATA Command

The DATA command is used as a reply of a GET_DATA command or when you receive a new transaction in the network and want to propagate it to your peers. Its payload is a string with the payload type and the transaction struct in hexadecimal in the following format: `payload_type:transaction_struct`, where the`payload_type` is either 'tx' or 'block' to indicate the type of the struct.

---

There are two steps from the beggining of the syncing until both peers are synced.

1. Find the highest timestamp where both peers are synced, i.e. they have all the transactions of each other from the genesis until this timestamp. This is called `synced_timestamp`
2. Sync the rest of timestamps until now, i.e. download all the transactions where `tx.timestamp > synced_timestamp`

### Find synced timestamp

To find the first timestamp where both are synced we iterate starting from the latest timestamp and decreasing it until we have a match. We define that two peers are synced in a given timestamp when the merkle tree of their tips are the same.

The merkle tree is calculated as the `sha256` of the hashes of all transactions in a given timestamp.

To optimize the search, it starts with an exponential search in the timestamp, where in each iteration we send a GET_TIPS command to the peer and compare its merkle tree with ours in this timestamp, until we have a match. In each iteration we decrease the timestamp (the decrease amount depends on the step of the exponential search) and we always hold the previous searched timestamp.

Then we do a binary search between the previous and current timestamp using the same process of sending a GET_TIPS command and comparing the merkle tree.

After both searches are finished we will have the highest timestamp where the peers are synced (defined as `synced_timestamp`), then we start downloading the transactions from this timestamp until the latest timestamp.


### Sync from synced_timestamp

In this step we iterate until we have downloaded all transactions until the latest timestamp. In each iteration we send a GET_NEXT command to get the hashes of the requested timestamp and offset. After receiving these hashes, we check which one of them the node still don't have and send a request to download them.

This request is made to a download coordinator, who will send a GET_DATA command only one time, replying a deferred that will be resolved when the corresponding DATA command arrives. This prevent from sending a GET_DATA command to all connected peers to transaction.

---

After theses two steps are completed, both peers are in sync. Every second the two steps above are executed again to validate that they are still synced. This check that two peers are synced is made with the following formula:

```
tx_storage.latest_timestamp - synced_timestamp <= sync_threshold
```

If this is true, the peers are synced. Where `tx_storage.latest_timestamp` is the highest timestamp of a transaction that your node has in the storage and the `sync_threshold` is a delta value of acceptance that they might be unsync. Right now this value is 60 seconds, i.e. two synced peers can have a difference of transactions and blocks at most of one minute.

## Real Time

During real time update both peers are already in sync, so every moment they receive a new transactions that were propagated in the network, they send it to all connected peers that are ready to receive it.

Each peer has two different queues, one for blocks that has a higher priority and must be propagated before the other, and another for transactions. To propagate a new transaction, the peer adds the data to one of the queues and wait for the queue do be consumed.

It does not propagate the data to all connected peers, the peer must be in the READY state and the `synced_timestamp` of the destination peer must be higher than the parents timestamps, i.e. the destination peer must have all the parents before download the new transaction.

After this validation, a DATA message is sent to the peer with the data of the new transaction.

# Drawbacks
[drawbacks]: #drawbacks

- Currently we are connecting to all the peers in the network, so with a small increase of full nodes connected the naive algorithms that are being used could generate a huge traffic in the network. Right now we check with all peers in the network our synced timestamp every second and also propagate new transactions to all of them.

# Future possibilities
[future-possibilities]: #future-possibilities

- In the real time sync we should not propagate the new transaction to all connected peers.
