# Summary

The goal of this document is to describe a new method of syncing with a Hathor node, using the reliable integrations feature and replace the current sync daemon on our production services

# Motivation

At present, our wallet-service sync daemon operates by utilizing the fullnode's WebSocket and HTTP APIs to maintain synchronization with the fullnode. With this approach, we have to manually handle reorganizations, making it not very maintenance friendly as we have to deal with all potential edge cases that are already handled in the fullnode, again in the daemon.
  
It was precisely with such applications in mind that we developed our '[Reliable Integrations](https://github.com/HathorNetwork/rfcs/blob/master/projects/reliable-integration/0001-high-level-design.md)' feature. The concept behind this feature is to implement a reliable, integrated event management system. Through this system, a series of events would be dispatched in the exact order of their occurrence, ensuring that if we follow this exact series of events, the same state will be obtained

# Reference-level explanation

In a nutshell, when the reliable integrations mechanism proposes a built-in event management system for applications interacting with a full-node. This system ensures events are sent in the order they occur, assigns unique IDs to each event, and offers both REST API and WebSocket access for event queries.

Given the synchronous nature of the event processing, I propose we design the daemon as a finite state machine as it can simplify the design by making it explicit which actions are valid in each state and how the system transitions between states.

Here is the proposed state diagram for handling fullnode's events:

![state](https://github.com/HathorNetwork/rfcs/assets/3586068/67af12d5-1afe-481d-841c-1685e2dc8f90)

There's an interactive version [here](https://stately.ai/registry/editor/f547b9f5-1987-4895-b3a1-be35e4daea54?machineId=bce0604d-b536-4086-8a7c-992e1754f54f)

### States

The machine starts by entering the `CONNECTING` state. It invokes an [`actor`](https://xstate.js.org/docs/guides/actors.html) that connects to the fullnode websocket feed and when it's done, it transitions to the `CONNECTED` state

#### `CONNECTED` 

**Entry Action**: As soon as the machine enters the `CONNECTED` state, it starts the streaming process by executing the `startStream` action, which will be documented in the `Actions` section of this document.
    
**Transitions**:
  - If a `DISCONNECT` event occurs (which can be sent from the websocket actor), the machine transitions back to the `CONNECTING` state.
**Guards**:
  - `checkPeerId` - The machine will only transition to `CONNECTED.idle` if the `checkPeerId` returns `true` (described in the Guards section)
  - `checkNetwork` - The machine will only transition to `CONNECTED.idle` if the `checkNetwork` returns `true` (described in the Guards section)

This machine has sub-states, the initial substate for this state is `idle`

#### `CONNECTED.idle`

This is the resting state where the machine awaits incoming events
    
**Transitions**:
- On receiving a `NEW_VERTEX_ACCEPTED` event:
  - **Guards**: if the `checkNew` guard (described in the Guards section) returns false, the machine will transition to the `CONNECTED.idle` state and ignore this event.
  - **Actions**: The machine stores the event using the `storeEvent` action.
  - **Target**: The machine transitions to the `handlingVertexAccepted` sub-state.
- On receiving a `VERTEX_METADATA_CHANGED` event:
  - **Guards**: if the `checkNew` guard (described in the Guards section) returns false, the machine will transition to the `CONNECTED.idle` state and ignore this event.
  - **Actions**: The machine stores the event using the `storeEvent` action.
  - **Target**: The machine transitions to the `handlingMetadataChanged` sub-state.
- On receiving a `LOAD_STARTED` event:
  - **Actions**: The machine stores the event with the `storeEvent` action.
  - **Target**: The machine transitions to the `success` sub-state.
- On receiving a `LOAD_FINISHED` event:
  - **Actions**: The machine stores the event with the `storeEvent` action.
  - **Target**: The machine transitions to the `success` sub-state.
- On receiving an `ACK` event:
  - **Actions**: The machine sends an acknowledgment using the `sendAck` action.
- On receiving any other event (represented by `'*'`):
  - **Actions**: The machine stores the event with the `storeEvent` action.
  - **Target**: The machine transitions to the `success` sub-state.


#### `CONNECTED.handlingVertexAccepted`

The machine will transition to this state to process a `NEW_VERTEX_ACCEPTED` event.

This event means that a new transaction passed the consensus mechanism on the fullnode and was accepted, so we need to include it on the wallet-service's database and update balances.

**Services**:
  - The machine invokes the `handleVertexAccepted` service, passing the event data. This service will be further explained in the `Services` section of this document. 
**Transitions**:
- If the service invocation completes successfully (`onDone`):
  - **Target**: The machine transitions to the `success` sub-state.
- If there's an error during the service invocation (`onError`):
  - **Target**: The machine transitions to the `error` sub-state.


#### `CONNECTED.handlingMetadataChanged`

The machine will transition to this state to process a `NEW_METADATA_ACCEPTED` event.

This event happens when the metadata of a transaction changes, this can happen for many reasons, but the ones we care about are:

- The transaction has been confirmed by a block
- The transaction was voided

**Services**:    
- The machine invokes (on entry) the `handleMetadataChanged` service, passing the event data. This will be further explained in the `Services` section of this document.
**Transitions**:
- If the service invocation completes successfully (`onDone`):
    - **Target**: The machine transitions to the `CONNECTED.success` sub-state.
- If there's an error during the service invocation (`onError`):
    - **Target**: The machine transitions to the `CONNECTED.error` sub-state.


#### `CONNECTED.success`

The machine will transition to this state when a event is successfully processed

**Entry Action**: As soon as the machine enters the `success` state, it sends an acknowledgment using the `sendAck` action.
**Transitions**:
  - The machine always transitions back to the `CONNECTED.idle` sub-state after performing the entry actions.

**Services**:
- The machine invokes (on entry) the `storeLastEvent` service, passing the event data and will only transition to `CONNECTED.idle` if the `event_id` was stored successfully

**Transitions**:
- If the service invocation completes successfully (`onDone`):
    - **Target**: The machine transitions to the `CONNECTED.idle` sub-state.
- If there's an error during the service invocation (`onError`):
    - **Target**: The machine transitions to the `CONNECTED.error` sub-state.

#### `CONNECTED.error`

The machine will transition to this state when an error is detected when event handling or invoking a `service`.

This is a `final` state, meaning that no further transitions can occur from this state and that the machine will need manual intervention to recover.

**Entry Action**: The machine logs the error using the `logError` action.


## Potential challenges

1. **Event Ordering and Consistency**: Given that events are sent in the order they occurred, it's crucial to ensure that the daemon maintains this order when processing events. This is especially important in scenarios like reorgs where the event order can significantly impact the wallet-service's state.
2. **Event Volume and Scalability**: Depending on the full-node's activity, the number of events generated can be significant and considering that we are processing those events one at a time, we should make sure that we are doing this as efficiently as possible
3. **Error Handling and Recovery**: There might be situations where the daemon encounters an error processing an event or loses connection to the full-node. We should make sure that we always have the latest successfully synced event id persisted and that we are rolling back changes that were unsuccessful to prevent invalid states


# Guide-level explanation

## Database design

Aside from the wallet-service existing database, we need to store metadata related event processing:

```sql
CREATE TABLE sync_metadata (
    id INT PRIMARY KEY,
    last_event_id INT NOT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

`id` - This column will always be '1', the idea is to always have a single row in this table to keep track of the last event received
`last_event_id` - This column will hold the last event that was successfully synced
`updated_at` -

This table should be created with the same migration mechanism the wallet-service uses. More about this in the monorepo section.

## Monorepo

We should merge both the wallet-service lambdas and the daemon in a single repository so they can share database migrations and we can have a better control over versions and database structures.

I suggest we do this using [yarn workspaces](https://classic.yarnpkg.com/lang/en/docs/workspaces/).


## Actors

> The [Actor model (opens new window)](https://en.wikipedia.org/wiki/Actor_model)is a mathematical model of message-based computation that simplifies how multiple "entities" (or "actors") communicate with each other. Actors communicate by sending messages (events) to each other. An actor's local state is private, unless it wishes to share it with another actor, by sending it as an event.
https://xstate.js.org/docs/guides/actors.html


#### WebSocketActor

The `WebSocketActor` will initiate and maintain a connection with the full-node's WebSocket feed.

We are using actors to have a bi-directional channel to the WebSocket feed from the machine, it will receive events and send those events to the machine, using the same type as received from the feed, so if new events are added in the future, they will be automatically emitted to the machine (and probably handled by the '\*' transition)

## Actions

#### sendAck

The `sendAck` action will publish an `ACK` message on the websocket connection acknowledging the last processed event.

```json
{
  "type": "ACK",
  "window_size": 1,
  "ack_event_id": lastEventId,
};
```

We will always send a `window_size` of 1 because we need to process events sequentially

## Guards

#### checkPeerId

This guard will if check the connected fullnode's peer-id is the same as the one set in the env `PEER_ID`.

#### checkNetwork

This guard will if check the connected fullnode's Network is the same as the one set in the env `NETWORK`.

#### checkNew

This guard will receive an event and check if the transaction has changed by comparing the data we have on the database with the data received on the event.

As an optimization, we will store an in-memory cache of recent transactions by holding a md5 hash of all the fields we are interested in (hash, voided_by length, first_block and height) so we can quickly detect if anything was changed without the need of a database query.

Here is an example of the method that will hash the data we're interested in

```javascript
const data = `${meta.hash}|${meta.voided_by.length > 0}|${meta.first_block}|${meta.height}`;
const hash = crypto.createHash('md5');
hash.update(data);
return hash.digest('hex');
```

The size of the LRU cache should be configured by an env variable.

## Services

#### storeLastEvent

This service will store the last successful `event_id` on the `sync_metadata` table.


#### handleVertexAccepted

This service handles the `NEW_VERTEX_ACCEPTED` event from the WebSocket feed, it means that the full-node's consensus mechanism accepted this transaction and added it to the blockchain, so we need to also include it in the wallet-service's database.

These are the steps that need to be taken when processing a new transaction:

1. Add the transaction to the `transaction` table
2. Add the transaction outputs to the `tx_outputs` table
3. Update the `tx_outputs` table, setting the `spent_by` column to the new transaction received if the `utxo` was used as an input.
4. Calculate the transaction balance for each address involved.
5. Update address balances (`address`, `address_balance`, `address_tx_history`) with the updated balances
6. Update wallet tables (`wallet_balance`, `wallet_tx_history`) with the transaction and updated balances.
7. Generate new addresses
8. If this is a block, unlock all `utxos` that were heightlocked at this height.
9. Use the transaction timestamp to unlock timelocked `utxos`
10. If this is a token creation transaction, add the token information on the `token` table
11. Increment the transactions count for each token involved in this transaction

*We already have those methods well tested in the wallet-service lambdas, we can just migrate them to the daemon.*

#### handleMetadataChanged

This service handles the `VERTEX_METADATA_CHANGED` event from the WebSocket feed, it will be emitted from the full-node on different situations:

1. The full-node consensus mechanism changed the metadata of this vertex before accepting it
2. This transaction has been confirmed by a block
3. This transaction has been voided

We need to detect and handle this event differently for each situation. We can detect the situation type by inspecting our own database and checking what changed.

`(1)`:
If we don't have this transaction on the database, it's the situation (1). We can just add it to the database by invoking the `handleVertexAccepted` service.

`(2)`:
We can do an `UPSERT` in the `transaction` database updating the `height` column. No balance re-calculation is needed here.

`(3)`:
If we do have the transaction on the database, we need to check both the `voided_by` attribute from the new `metadata` (that came on the event) and the `voided` column from the wallet-service's `transaction` table. If the `voided` column in the database is `FALSE`, we need to handle a voided transaction. These are the steps that need to be taken:

1. Calculate the address balance for each address involved in this transaction
2. Update address balances (`address`, `address_balance`, `address_tx_history`) with the updated balances
3. Update wallet tables (`wallet_balance`, `wallet_tx_history`) with the transaction and updated balances.
4. Decrement the transactions count for each token involved in this transaction

*We already have those methods well tested in the wallet-service lambdas, we can just migrate them to the daemon.*


# Future Considerations

## Simplified balance calculation

There is a potential optimization to handling events by using the `group_id` attribute in our advantage for processing events in parallel. The approach can be summarized as follows:

1. Group events by group_id.
1. For each group of events, start a new database transaction by executing BEGIN TRANSACTION.
1. Process each event individually. When an output is unspent, add an entry to the UTXO table. If an output is spent, remove the corresponding UTXO.
  1. Note: Inputs or parents related actions are not necessary in this step
  1. Events can be processed in parallel if and only if they're from different transactions.
1. After processing all events in a group:
  1. Generate new addresses.
  1. Unlock any time-locked UTXOs.
  1. Recalculate the balance.
  1. Commit the changes to the database using COMMIT.

We decided not to implement it in the initial version because we don't yet have a mechanism to know when a group has started or ended.
