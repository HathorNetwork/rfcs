# Summary

A new connection class that streams events from the fullnode event queue.
# Motivation

The event queue is a way to sequentially stream events, making sure we do not miss any transaction or transaction update, we can also continue where we left in case of any connection issues, currently any connection issue makes it so we have to sync the entire wallet history from the beggining.
# Guide-level explanation

The fullnode's event queue (a.k.a. reliable integration) has 2 ways to stream events, a websocket and a http api (not server-side events (SSE), just polling events with an event id based pagination).
We will focus on the websocket implementation but we can later create another class that uses the http api.

## Events

The events from the event queue are not filtered in any way, meaning we have to implement our own filter.
The events that represent new transactions or blocks is called `NEW_VERTEX_ACCEPTED` and any updates on the transaction data is called `VERTEX_METADATA_CHANGED` these 2 events come with the current transaction data, we can use this data to filter for transaction of our wallet.

## Address subscription

The current implementation uses the fullnode pubsub to listen only for addresses of our wallet, meaning that during startup we send a subscribe command for each address of our wallet.
Since the events will be filtered locally we can instead use the `subscribeAddresses` method to create a local list of addresses being listened to and use this list to filter the events.

The "list of addresses" will be an object where the address is the key, since determining if an object has a key is O(1) we can ensure that this does not become a bottleneck for wallets with many addresses.
Alternatively, we could use the storage `isAddressMine` method which is already O(1).

## Event streaming

To get the full state of the wallet we would need to stream all events of the fullnode, but we can still use the address history api to fetch the balance and transactions of our addresses and start listening the newer events.

The best way to achieve this is to use the event api to fetch a single event, this will come with the latest event id.
Example response of `GET ${FULLNODE_URL}/v1a/event?size=1`

```json
{
    "events": [
        {
            "peer_id": "ca084565aa4ac6f84c452cb0085c1bc03e64317b64e0761f119b389c34fcfede",
            "id": 0,
            "timestamp": 1686186579.306944,
            "type": "LOAD_STARTED",
            "data": {},
            "group_id": null
        }
    ],
    "latest_event_id": 9038
}
```

Then we save `latest_event_id` and sync the history with the address history api.
Once we have the wallet on the current state we can start streaming from the `latest_event_id`.
There can be transactions arriving during this process which would mean we add them during the history sync and during the event streaming, but this issue does not affect the balance or how we process the history.

## Best block update

The fullnode pubsub sends updates whenever the best chain height changes, this is so our wallets can unlock our block rewards, the event queue does not send an update like this but we receive all block transactions as events, meaning we can listen for any transaction with `version` 0 or 3 (block or merged mining block) and with the metadata `voided_by` as `null` (this is because if a block is not voided, it is on the main chain) and derive the best chain height.

We will always expect the latest unvoided block to be the best chain newest block since during re-orgs (where the best chain changes) we will receive updates and the new best chain will be updated with `voided_by` as `null`.

## EventQueueConnection class

The `EventQueueConnection` class will manage a websocket instance to the event queue api and emit a `wallet-update` event, this is to keep compatibility with the existing `Connection` class.

The `wallet-update` event will work with the schema:

```ts
interface WalletUpdateEvent {
	type: 'wallet:address_history',
	history: IHistoryTx,
}

// Where IHistoryTx is defined as:

interface IHistoryTx {
  tx_id: string;
  signalBits: number;
  version: number;
  weight: number;
  timestamp: number;
  is_voided: boolean;
  nonce: number,
  inputs: IHistoryInput[];
  outputs: IHistoryOutput[];
  parents: string[];
  token_name?: string;
  token_symbol?: string;
  tokens: string[];
  height?: number;
}

export interface IHistoryInput {
  value: number;
  token_data: number;
  script: string;
  decoded: IHistoryOutputDecoded;
  token: string;
  tx_id: string;
  index: number;
}

export interface IHistoryOutputDecoded {
  type?: string;
  address?: string;
  timelock?: number | null;
  data?: string;
}

export interface IHistoryOutput {
  value: number;
  token_data: number;
  script: string;
  decoded: IHistoryOutputDecoded;
  token: string;
  spent_by: string | null;
}
```

The transaction data from the event queue is in a different format as described below:

```ts
interface EventQueueTxData {
  hash: string;
  version: number;
  weight: number;
  timestamp: number;
  nonce?: number;
  inputs: EventQueueTxInput[];
  outputs: EventQueueTxOutput[];
  parents: string[];
  token_name?: string;
  token_symbol?: string;
  tokens: string[];
  metadata: EventQueueTxMetadata;
  aux_pow?: string;
}

interface EventQueueTxMetadata {
  hash: string;
  spent_outputs: EventQueueSpentOutput[];
  conflict_with: string[];
  voided_by: string[];
  received_by: string[];
  children: string[];
  twins: string[];
  accumulated_weight: number;
  score: number;
  first_block?: string;
  height: number;
  validation: string;
}

interface EventQueueTxInput {
  tx_id: string;
  index: number;
  data: string;
}

interface EventQueueTxOutput {
  value: number;
  script: string;
  token_data: number;
}

interface EventQueueSpentOutput {
  index: number;
  tx_ids: string[];
}
```


### Data conversion process

To keep compatibility with the current `Connection` class we need to convert the data with the following process:

1. Convert `hash` to `tx_id`
2. Assert `nonce` is valid
3. Assign `signalBits` from `version`
4. Assign `is_voided` from metadata's `voided_by`
5. Assign `height` from metadata's height
6. Convert outputs
7. Convert inputs
8. remove `metadata` and `aux_pow` fields

Process to convert outputs:

1. Derive `token` from `token_data` and `tx.tokens`
2. Derive `spent_by` from the output index and `tx.metadata.spent_outputs`
3. Derive `decoded` from the script

Process to convert inputs:

1. Try to find the transaction with the input's `tx_id` in storage
	1. If not found we must fetch the tx from the fullnode api
2. Assign `value`, `token_data`, `script` and `token` from the spent output
3. Derive `decoded` from the script.

Now that the data matches the current websocket transaction we can emit the data and all processes to manage history from the facade will work as intended.
