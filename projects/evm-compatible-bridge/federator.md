- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary

This is the description of how the federator service will coordinate the events on both sides of the bridge.

# Guide-level explanation

The federator service will keep track of events on both sides of the bridge and vote on the actions to be taken.
This will be done by watching events on the federation contract on the EVM chain and polling events from the coordinator service on Hathor.

Since Hathor does not currently have a smart contract implementation, we will use a MultiSig address to check that a certain number of federators have consented to any transaction.
To avoid separate instances of the MultiSig wallet to propose conflicting transactions, we will use a coordinator service to choose the utxos and collect signatures from the federators.

Each federator will have 2 loops, one to check events on the EVM compatible chain (i.e. EVM loop) and another to check events on the coordinator service (i.e. polling loop).
The EVM loop will get the events from the chain, this loop will have events of transaction requests from the EVM compatible chain to Hathor and confirmations on transactions.
The polling loop will check the coordinator service polling api and vote to perform the actions as requested.

## Federator service

### EVM loop

We need to get events from the bridge contract, wait a pre-configured number of blocks (to increase the confirmation level of the transaction) and send the transaction on Hathor as requested.
There should be 3 types of events:

- A request to cross the token from the EVM chain to Hathor
  - If the token is native to the EVM chain, an equivalent Hathor token should already exist and the MultiSig wallet should own the authority utxos to it.
  - If the token is native to the Hathor chain, the equivalent token is already burned and we need to unlock the tokens on Hathor.
  - The loop will have to use the coordinator service to send the amount of tokens requested from the MultiSig wallet to the destination address.
- A request to support a new token native to the EVM chain.
  - We need to check if this is a duplicate request
  - We need to use the coordinator service to create of the equivalent token on Hathor.
  - We need to use the bridge smart contract to save the Hathor equivalent token uid once the token is created.
- Confirmation that an operation is complete
  - This way we can clean any state pertaining to this crossing of tokens and mark it as complete.
  - If the crossing was originated in Hathor, we need to send a request to the coordinator service so it knows the crossing is over.

### Polling loop

We will pool for events, which are requests for voting, and vote on them.
If the event requires a vote on the smart contract, we will send a transaction to the EVM chain with the vote.
If the event requires a vote on the MultiSig wallet, we will send a message to the coordinator service with a signature.

The federator will be responsible to check the data on the event, it will have its own headless wallet to check the information from the Hathor network and it can read the data from the EVM chain.

## Coordinator service

The coordinator service will serve the function of the smart contract in the Hathor side.
It will listen for transactions on the MultiSig wallet, keep a comprehensive persistent database so we do not process the same transaction twice and will gather signatures from the federation.
It will be started with the MultiSig wallet config but it will not be a participant of the wallet.

The service will download the history of transactions of the MultiSig wallet and process all transactions.
If a transaction was already processed, the federation contract should have it in its state, so we can safely ignore it.

The coordinator service will provide a REST API for the federators and admins to interact with it.

### Processing Hathor transactions

When first starting the process it will iterate over all transactions in the MultiSig wallet and process them.
After this, each new message will be enqueued and processed in order of arrival.
Obs: The headless wallet offers a message queue extension to enqueue events e.g. new transactions.

First we need to check the transaction is compliant with the request format if not, we can save it as processed and continue.
If the transaction is already on tha database we can skip it, if not we need to check the federation contract to see if it was already processed.
Obs: This read operation on the smart contract does not require a transaction to be sent, we can read the state from the EVM chain with 0 cost.

The request format for a transaction is very simple:

- The first output should be a data output, the data should be the destination address on the EVM.
- There should only be 1 token being sent to the MultiSig wallet.

To check this format we will call the headless api to check the transaction data and validate based on the response.

Since this transaction was made from Hathor it will be saved as a Hathor request on the database with the following fields:

- Hathor tx_id
  - With the tx_id, so the federators can inspect the contents.
- Hathor token uid
  - The equivalent EVM token can be found on the smart contract storage (it will be created if this is the first time we see this token)
- Amount of tokens being sent to the MultiSig wallet
- destination address (on EVM chain)
- Height
  - Current height of the network when this was received.
  - Transactions in Hathor do not have a height, so this is the height of blocks when the transaction was received.

With this we can safely wait for 5 blocks on Hathor to have passed so we can create the proposal.
This proposal will be available for the polling API as soon as it is created.

### API

#### GET /events

This is the polling api, an event is a request to collect signatures, this means there are events to collect signatures in the MultiSig wallet and events to collect signatures on the federation contract.

Both kinds of events will be returned by this endpoint, the federator will be responsible to make the appropriate calls to the MultiSig wallet or the federation contract.

The response will contain a list of events and pagination data.

```ts
type HathorTxProposal = {
  eventOrigin: 'htr' | 'evm' | 'admin',
  type: 'create' | 'mint' | 'send' | 'melt', // This indicates the action on the Hathor MultiSig wallet
  proposalId: number,
  txHex: string,
  // The event will have 1 of the following fields
  evmRequestId?: number,
  htrRequestId?: number,
  adminRequestId?: number,
};

type EvmVoteProposal = {
  eventOrigin: 'htr' | 'evm' | 'admin',
  type: 'send', // The contract will know to mint or unlock based on the token address
  proposalId: number,
  // The event will have 1 of the following fields
  evmRequestId?: number, // Request for fee collection
  htrRequestId?: number, // Request to cross tokens
  adminRequestId?: number, // For refunds
};

type Event = HathorTxProposal | EvmVoteProposal;

type ApiResponse = {
  events: Event[],
  hasMore: boolean,
};
```

#### POST /sign/{proposal_id}

The proposal id is referring a Hathor tx proposal where the federator has signed and this will collect the signatures.

Once enough signatures are collected, the transaction will be send to the network.

The endpoint will be expecting the fields:
  
```ts
type RequestBody = {
  signature: string,
  publicKey: string,
};
```

The public key is required so we can count how many distinct signatures were collected.

#### POST /request/htr

The federators will call this endpoint when a new Hathor request is made, once the event reaches the federators they will call the federation contract to fulfill this request.

The endpoint will be expecting the fields:
  
```ts
type RequestBody = {
  txId: string,
  tokenUid: string,
  amount: string,
  destinationAddress: string,
};
```

#### POST /request/evm

The federators will call this endpoint when an event is received in the EVM loop.

The endpoint will be expecting the fields:
  
```ts
type EvmTokenId = {
  address: string,
  tokenId?: number,
}

type RequestBody = {
  kind: 'cross' | 'new_token',
  blockHash: string,
  txHash: string,
  evmTokenId: EvmTokenId,
  // These will only be used for cross requests
  amount?: string,
  destinationAddress?: string, // Destination address on Hathor
};
```

#### POST /fulfill/{htr_request_id}

When a request is fulfilled, the federator will call this endpoint to inform the API that the request was fulfilled.

Hathor requests can only end when the tokens are sent or minted on the EVM chain.
This means this endpoint will be called when the bridge contract has sent the event to confirm that the tokens were minted or unlocked.

```ts
type RequestBody = {
  // Data to identify the transaction that fulfilled the request
  blockHash: string,
  txHash: string,
  logIndex: number, // Of the event that confirmed the fulfillment
};
```

#### POST /fulfill/{evm_request_id}

Requests originated in the EVM chain can only end when the Hathor request is sent to the network.
This means this endpoint will be called when the coordinator has sent the transaction to the network.

#### POST /fulfill/{admin_request_id}

Some admin requests are fulfilled when an action is made on the bridge contract (i.e. in the EVM chain) this should cause an event to be emitted and the federator EVM loop can call this API to mark the request as fulfilled.

```ts
type RequestBody = {
  // Data to identify the transaction that fulfilled the request
  blockHash: string,
  txHash: string,
  logIndex: number, // Of the event that confirmed the fulfillment
};
```

#### Admin requests

The admin can request refunds and sending transactions from the MultiSig wallet and the contract to any user address.
This is to fix any mistakes that might happen.

- `POST /request/admin/refund/evm`
  - Requires the block hash and transaction of the user request for a token cross.
- `POST /request/admin/refund/htr`
  - Requires the transaction id of the user request.
- `POST /request/admin/send/htr`
  - Requires the transaction hex to be sent.
- `POST /request/admin/send/evm`
  - Requires the destination address, token address or token id.
- `POST` /request/fee
  - Requires the destination address, token address or token id.
  - Similar to `/request/send/evm` but the federation contract will call a fee collection method on the bridge contract.

#### Fulfilling requests without API calls

Requests originated on the EVM chain or admin requests that are fulfilled by sending a transaction on the Hathor network do not require an API call since the coordinator service will be sending the transaction to the network.
This means it can also save the data on the database without calling its own API.

### Database

#### Tables: Hathor requests

| id | tx_id | token_uid | amount | destination_address | height | fulfilled |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Integer | String(64) | String(64) | BigInt | String | Integer | Boolean |

#### Tables: EVM requests

| id | block_hash | tx_hash | log_index | evm_token_id | amount | destination_address | height | fulfilled |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Integer | String(64) | String(64) | Integer | String(42) | BigInt | String | Integer | Boolean |

#### Tables: Hathor tx proposals

| id | type | tx_hex | fulfilled | admin_request_id | hathor_request_id | evm_request_id |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Integer | String | String | Boolean | Integer | Integer | Integer |

#### Tables: Signatures

| id | proposal_id | signature | public_key |
| :---: | :---: | :---: | :---: |
| Integer | Integer | String | String |

#### Tables: Admin requests

The following `request_id` fields are a foreign key to the admin request table, which is a table with a single column `id` of type `Integer`.
This is to ensure the admin request id is unique across all tables.

##### EVM Refunds

| request_id | block_hash | tx_hash |
| :---: | :---: | :---: |
| Integer | String(64) | String(64) |

##### HTR Refunds

| request_id | tx_id |
| :---: | :---: |
| Integer | String(64) |

##### HTR Send

| request_id | tx_hex |
| :---: | :---: |
| Integer | Text |

##### EVM Send

| request_id | destination_address | evm_token_address | evm_token_id |
| :---: | :---: | :---: | :---: |
| Integer | String(42) | String(42) | Integer |

##### Fee collection

| request_id | destination_address | evm_token_address | evm_token_id |
| :---: | :---: | :---: | :---: |
| Integer | String(42) | String(42) | Integer |
