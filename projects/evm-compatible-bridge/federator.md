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
There should be 2 types of events:

- A request to cross the token from the EVM chain to Hathor
  - If the token is native to the EVM chain, an equivalent Hathor token should already exist and the MultiSig wallet should own the authority utxos to it.
  - If the token is native to the Hathor chain, the equivalent token is already burned and we need to unlock the tokens on Hathor.
  - The loop will have to use the coordinator service to send the amount of tokens requested from the MultiSig wallet to the destination address.
- Confirmation that an operation is complete
  - This way we can clean any state pertaining to this crossing of tokens and mark it as complete.
  - If the crossing was originated in Hathor, we need to send a request to the coordinator service so it knows the crossing is over.

### Polling loop

We will pool for events, which are requests for voting, and vote on them.
If the event requires a vote on the smart contract, we will send a transaction to the EVM chain with the vote.
If the event requires a vote on the MultiSig wallet, we will send a message to the coordinator service with a signature.

The federator will be responsible to check the data on the event, it will have its own headless wallet to check the information from the Hathor network and it can read the data from the EVM chain.

### Transaction inspection

Each federator will inspect the transactions before signing them, checks they do on each transaction will change depending on the intended action.

Crossing tokens from Hathor to EVM (token native to EVM):

- Check that the transaction is a melt operation.
- Check the wallet balance has enougth tokens to melt.
- Check that the amount to melt is equal to the amount crossed.

Crossing tokens from Hathor to EVM (token native to Hathor):

- This operation does not require the federation to send a transaction in Hathor, so the validations will be done in the smart contract.

Crossing tokens from EVM to Hathor (token native to EVM):

- Check that the transaction is a mint operation.
- Check the wallet balance has enougth HTR to mint.
- Check that the amount to mint is equal to the amount requested minus fees.

Crossing tokens from EVM to Hathor (token native to Hathor):

- Check that the token amount minus the fee is being sent to the destination address.
- Check that the fee amount is sent to the admin address.

## Coordinator service

The coordinator service will serve the function of the smart contract in the Hathor side.
It will listen for transactions on the MultiSig wallet, keep a comprehensive persistent database and will gather signatures from the federation.
It will be started with the MultiSig wallet config but it will not be a participant of the wallet.

During the startup we need to check all existing transactions on the MultiSig address, if the transaction was already processed we can safely ignore it.
After the startup we can start to listen for transactions using the queue plugin which enqueues events like new transactions made to one of our addresses.

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
  - Height of the first block that confirms the transaction.
  - Transactions in Hathor do not have a height but we can use the `first_block_height` metadata.

With this we can safely wait for 5 blocks on Hathor to have passed so we can create the proposal.
There is an event `best-block-update` which tells the current height of the network.
This proposal will be available for the polling API as soon as it is created.

### Coordinator service API

#### GET /events

This is the polling api, an event is a request to collect signatures, this means there are events to collect signatures in the MultiSig wallet and events to collect signatures on the federation contract.

Both kinds of events will be returned by this endpoint, the federator will be responsible to make the appropriate calls to the coordinator service or the federation contract.

The response will contain a list of events and pagination data.

```ts
type Event = {
  eventOrigin: 'htr' | 'evm' | 'admin',
  type: 'mint' | 'melt' | 'send', // This indicates the action on the Hathor MultiSig wallet
  proposalId: number,
  txHex: string,
  // The event will have 1 of the following fields
  evmRequestId?: number,
  htrRequestId?: number,
  adminRequestId?: number,
};

type ApiResponse = {
  events: Event[],
  hasMore: boolean,
};
```

#### POST /sign/{proposal_id}

The proposal id is referring a Hathor tx proposal where the federator has signed and this api will be used to collect the signatures.

Once enough signatures are collected, the transaction will be sent to the network.

The endpoint will be expecting the fields:

```ts
type RequestBody = {
  signature: string,
  publicKey: string,
};
```

The public key is required so we can count how many distinct signatures were collected.

#### POST /request/evm

The federators will call this endpoint when an event is received in the EVM loop.
The coordinator will check the smart contract to validate the information received by the federation.

The endpoint will be expecting the fields:

```ts
type RequestBody = {
  blockHash: string,
  txHash: string,
  evmTokenAddr: address,
  amount: string,
  destinationAddress: string,
};
```

#### POST /fulfill/htr/{htr_request_id}

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

#### POST /fulfill/admin/{admin_request_id}

Admin requests are fulfilled when a transaction is sent in Hathor network and this API should be used to mark the request as fulfilled.

```ts
type RequestBody = {
  txHash: string,
  htrTokenUid: string,
  amount: string,
  destinationAddress: string,
};
```

#### Admin requests

The admin should be able to request a transaction from the MultiSig wallet to any user address.
This is to fix any mistakes that might happen, like refunding a transaction.

- `POST /request/admin/refund`
  - Requires the transaction id of the user request.

#### Coordinator fulfilling requests

When a request is fulfilled by a transaction being sent on Hathor, it means that the coordinator will be aware of the fulfillment since it will be the one sending the transaction.
This means that when the confirmation of the request is received by the coordinator it will be responsible to mark the request as fulfilled.

### Database

#### Tables: Hathor requests

| id | tx_id | token_uid | amount | destination_address | height | fulfilled |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Integer | String(64) | String(64) | BigInt | String | Integer | Boolean |

#### Tables: EVM requests

| id | block_hash | tx_hash | log_index | evm_token_address | amount | destination_address | height | fulfilled |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Integer | String(64) | String(64) | Integer | String(42) | BigInt | String | Integer | Boolean |

#### Tables: Hathor tx proposals

| id | type | tx_hex | fulfilled | admin_request_id | evm_request_id |
| :---: | :---: | :---: | :---: | :---: | :---: |
| Integer | String | String | Boolean | Integer | Integer |


Obs: the `hathor_request_id` is not present on this table because a transaction originated in Hathor cannot request a transaction to be made on Hathor network.

#### Tables: EVM tx proposals

| id | evm_request_id | fulfilled |
| :---: | :---: | :---: |
| Integer | Integer | Boolean |

#### Tables: Signatures

| id | proposal_id | signature | public_key |
| :---: | :---: | :---: | :---: |
| Integer | Integer | String | String |

#### Tables: Admin Refund Requests

| id | tx_id |
| :---: | :---: |
| Integer | String(64) |


### Supporting new tokens

#### Hathor native token

The admin should make a transaction on the EVM contract passing the Hathor token data to create the new side token and give the bridge address the permission to mint and melt this new token.

This new side token will have an address, this address will be entered in the mapping along with the token uid making this token a valid token to be used.
Then the admin should add this token on the allowed tokens, this will make the bridge allowed to process transactions with this token.

#### EVM native tokens

The admin should create the equivalent token on Hathor, melt all available tokens then send the authorities to the federation MultiSig address.
This will make the federation capable of minting and melting the token but the token will only be allowed when its information is added to the allowed tokens contract by the admin.

