## Problem

Currently, applications that want to interact with the full-node must write its own sync algorithm and handle all use cases (Like reorganization). This algorithm can become very complex and consume several days of development.

There are two alternatives when applications want to interact with the full node:

- Initialize a bidirectional communication via websocket.
- Call the REST APIs.

However, the main concerns with these approaches are:

- When a reorganization happens, the applications must know how to query all the affected txs and update them on their databases.
- There is no guarantee on the order the events happened.
- When the websocket is disconnected, events might be lost and a resync must be executed.

## Solution

To tackle the problems presented above, we must implement a built-in event management system, where all events will be sent in the order they occurred. This system have the following requirements:

1. Detect events that are important to applications. (Look for `Event Types` below).
1. Persist each event and give it an unique incremental ID.
1. Give users an REST API and Websocket connection to query for events

To set-up this system, the user must provide, during the full-node initialization, the `--enable-event-queue` flag.

Due to the necessary flags and the events that can be emitted, we will provide a document on how to understand this new mechanism.

This project, however, does **NOT** intend to eliminate the necessity of a sync algorithm, but to make it much more simpler.

## Out of Scope

These features will not be part of the first phase of this project:

- Sync pause/resume
  - An API to manipulate the sync algorithm.
- Event filter
  - Give users the choice to receive only a subset of events, according to some criteria.
- The `--flush-events` flag. By default, all events are retained. In the future, the user could provide a `--flush-events` flag to enable the flushing of events after each event is sent to the client. 

## Event generation during the full-node cycle

Considering this full-node cycle:

![Full-Node-Cycle drawio](https://user-images.githubusercontent.com/3287486/179076836-ab6ae036-b0c8-449c-a582-2f29ca58ab2b.png)

Where:
- `Load` is the period right after the full-node is started, where the local database is read.
- `Sync` is the period where `Load` is finished and the full-node continuously receive/send txs to other peers until full node is stopped.

By default, the events generated during load will be emitted. If user does not want this behavior, it must provide `--skip-load-events` flag.

## Flow

![Event Flow drawio](https://user-images.githubusercontent.com/3287486/178827445-cc4975f7-3b75-4bc9-a79c-e1042a8a64be.png)

## API

To retrieve the events, an API will be provided:

`GET /event?last_ack_event_id=:last_ack_event_id&size=:size`

The maximum value for `size` will be 1000, to prevent DoS attacks, and the default value will be 100. Those should be configured via settings.

If `last_ack_event_id` is not provided, the first event on the database will be returned.

The result will be an array of events. Each entry will have the data format described on the `Data Format` section below. Also, a `latest_event_id` field will be returned, so the client will know how many events ahead are already available.

## WebSocket

In addition to the REST API, a similar WebSocket API will be available. The client will start the communication also providing the `last_ack_event_id` and an `window_size`. The server will keep the connection open and start sending streaming events to the client.

Each message will be a single event. The server will keep sending events while there are events available and there's `window_size` available. The client can send an ACK message at any time to either update the `ack_event_id` and/or the `window_size`, effectively having full control of the flow of events. Each entry will have the data format described on the `Data Format` section below. Also, a `latest_event_id` field will be returned, so the client will know how many events ahead are already available.

## Storage

To avoid adding new dependencies, the RocksDB, which is already implemented for `tx_storage`, will be used to persist the events. RocksDB is a key-value store. The key will be the event id, and the value will be the whole event. This way, querying a range of keys must be cheap (seek of O(log(n)) and each next is O(1)).

To serialize the data, we will transform the whole event into a JSON object and store as bytes on the RocksDB. A simple `json.decode` will be sufficient to retrieve the encoded data.

## Retention Policy

By default, all events will be stored, given this feature is enable via the `--enable-event-queue` flag. In future phases of the project, a `--flush-events` flag will be implemented to control the retention policy.

## Data Format

All events will have the following structure:

```
{
    full_node_uid: string, // Full node UID, because different full nodes can have different sequences of events
    id: uint64, // Event order
    timestamp: int, // Timestamp in which the event was emitted. This will follow the unix_timestamp format
    type: enum, // One of the event types
    group_id: uint64, // Used to link events. For example, many TX_METADATA_CHANGED will have the same group_id when they belong to the same reorg process
    data: {}, // Variable for event type. Check Event Types section below
}
```

## Data types:

```
tx_input: {
    value: int,
    token_data: int,
    script: string,
    index: int,
    tx_id: bytes,
}
```

```
tx_output: {
    value: int,
    script: bytes,
    token_data: int
}
```

```
token: {
    uid:  string,
    name: string,
    symbol: string
}
```

```
tx: {
    hash: string,
    nonce: int,
    timestamp: long,
    version: int,
    weight: float,
    inputs: tx_input[],
    outputs: tx_output[],
    parents: string[],
    tokens: token[],
    token_name: string,
    token_symbol: string,
    metadata: tx_metadata
}
```

```
spent_outputs: {
    spent_output: spent_output[]
}
```

```
spent_output: {
    index: int,
    tx_ids: string[]
}
```

```
tx_metadata: {
    hash: string,
    spent_outputs: spent_outputs[],
    conflict_with: string[]
    voided_by: string[]
    received_by: int[]
    children: string[]
    twins: string[]
    accumulated_weight: float
    score: float
    first_block: string or null
    height: int
    validation: string
}
```

```
tx_metadata_changed: {
    old_value: tx_metadata,
    new_value: tx_metadata
}
```

```
reorg: {
    reorg_size: int,
    previous_best_block: string, //hash of the block
    new_best_block: string //hash of the block
}
```

## Event Types
- LOAD_STARTED
- LOAD_FINISHED
- NEW_TX_ACCEPTED
- NEW_TX_VOIDED
- NEW_BEST_BLOCK_FOUND
- NEW_ORPHAN_BLOCK_FOUND
- REORG_STARTED
- REORG_FINISHED
- TX_METADATA_CHANGED
- BLOCK_METADATA_CHANGED

### LOAD_STARTED

It will be triggered when the full-node is initializing and the the full node is reading locally from the dabatase, at the same time of `MANAGER_ON_START` Hathor event [here](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/manager.py#L248). It should have an empty body. 

### LOAD_FINISHED

It will be triggered when the full-node is ready to establish new connections, sync, and exchange transactions, at the same that when the manager state changes to `READY` [here](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/manager.py#L513). Other events will be triggered ONLY after this one.

### NEW_TX_ACCEPTED

It will be triggered when the transaction is synced, and the consensus algorithm immediately identifies it as an accepted TX that can be placed in the mempool. `tx` datatype is going to be sent. We will reuse the `NETWORK_NEW_TX_ACCEPTED` Hathor Event that is already triggered.

### NEW_TX_VOIDED

It will be triggered inside [this method](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/consensus.py#L967) when the transaction is received and synced, but the consensus immediately identifies it as a voided TX (As long as a reorg is not in progress). `tx` datatype is going to be sent.

### NEW_BEST_BLOCK_FOUND

It will be triggered when a block is found, received by the full node and immediately identified as a valid block by the consensus algorithm. We will reuse the `NETWORK_NEW_TX_ACCEPTED` Hathor event that is already triggered. In addition, it will trigger ```TX_METADATA_CHANGED``` events for all transactions being directly confirmed by this block (Changing the ```first_block``` information). `tx` datatype is going to  be sent.

### NEW_ORPHAN_BLOCK_FOUND

It will be triggered inside [this method](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/consensus.py#L443) when a block is found, received by the full node, but immediately identified as a block that does not belong to the best chain. `tx` datatype is going to be sent.

### REORG_STARTED

It will be trigger right above [this line](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/consensus.py#L293), indicating that the best chain has changed. It will trigger the necessary ```TX_METADATA_CHANGED``` and ```BLOCK_METADATA_CHANGED``` events to void/execute them. `reorg` datatype is going to be sent.

### REORG_FINISHED

It will be triggered right below [this line (outside the for)](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/consensus.py#L106) if a `REORG_STARTED` had been triggered previously, indicating that the reorg (i.e. a new best chain was found) was completed and all the necessary metadata update was included between ```REORG_STARTED``` and this event.

### TX_METADADATA_CHANGED

Initially, we will trigger this event for two use cases:

- When a best block is found. All transactions will change its ```first_block``` metadata, which will be propagated through this event. This event will be propagated at [this point](https://github.com/HathorNetwork/hathor-core/blob/d39fd0d9515c72b5d68b36594294dbd7a155e365/hathor/consensus.py#L579) 
- When a reorg happens. This can trigger multiple transactions and blocks being changed to voided/executed. This will be detected on `mark_as_voided` functions inside `consensus.py` file (As long as consensus context finds that a reorg is happening). 

Data type `tx_metadata_changed` is going to be sent. Only the affected attributes, along with the hash, will be sent.

## Scenarios

- Two transactions are accepted into the mempool, and a block on the best chain is found to confirm those transactions

    1. NEW_TX_ACCEPTED (Tx 1)
    1. NEW_TX_ACCEPTED (Tx 2)
    1. NEW_BEST_BLOCK_FOUND (Block 1)
    1. TX_METADADATA_CHANGED (Changing the `first_block` of `Tx 1` to `Block 1`)
    1. TX_METADADATA_CHANGED (Changing the `first_block` of `Tx 2` to `Block 1`)

- Two transactions are accepted into the mempool. A block on the best chain is found to confirm those transactions, but a new block on a side chain arrives and becomes the best chain. The transactions are confirmed by this new block.

    1. NEW_TX_ACCEPTED (Tx 1)
    1. NEW_TX_ACCEPTED (Tx 2)
    1. NEW_BEST_BLOCK_FOUND (Block 1)
    1. TX_METADADATA_CHANGED (Changing the `first_block` of `Tx 2` to `Block 1`)
    1. TX_METADADATA_CHANGED (Changing the `first_block` of `Tx 1` to `Block 1`)
    1. REORG_STARTED
    1. NEW_BEST_BLOCK_FOUND (Block 2)
    1. BLOCK_METADATA_CHANGED (Changing the `voided_by` of `Block 1`)
    1. TX_METADATA_CHANGED (Changing the `first_block` of `Tx 1` to `Block 2`)
    1. TX_METADATA_CHANGED (Changing the `first_block` of `Tx 2` to `Block 2`)
    1. REORG_FINISHED

## Integration Tests

We will provide test cases with sequences of events for each scenario. This will help application to integrate with this new mechanism. 


## Task Breakdown and Effort

- [x] Proof of Concept (2 dev-days)
- [x] Low-level design (2 dev-days)
- [x] Change manager to handle the new flag (2 dev-days)
- [x] Make the REORG event be emitted during consensus (2 dev-days)
- [x] Emit event when tx/block is voided (2 dev-days)
- [x] Emit event for metadata changes (2 dev-days)
- [x] Create RocksDB event column family (2 dev-days)
- [x] Implement an event management layer, where events will be detected and persisted on RocksDB (3 dev-days)
- [x] Implement event persistence layer on RocksDB (3 dev-days)
- [x] Implement WebSocket API (2 dev-days)
- [x] Implement `GET /event` REST API  (1 dev-day)
- [ ] Implement `--skip-load-events` flag  (2 dev-days)
- [ ] Doc with user instructions  (1.5 dev-days)
- [ ] Build testing cases for integrations  (2 dev-days)

**Total: 28.5 dev-days**
