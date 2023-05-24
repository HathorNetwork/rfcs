- Feature Name: evm_compatible_bridge
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

This is the description of how the federation service will coordinate the events on both sides of the bridge.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The federator is a service that polls a main chain for events on the smart contract and votes on the sidechain to send tokens to users (and vice versa).
Since Hathor does not currently have a smart contract implementation, we will use a MultiSig address for the same effect. The problem is that for a transaction to be sent the MultiSig participants must sign the transaction and coordinate on which inputs to spend.

For this purpose we will be introducing a coordinator service to construct the transaction and choose which inputs should be spent.
Each federator will have 2 loops, one to check events on the EVM compatible chain (i.e. EVM loop) and another to check events on the coordinator service (i.e. polling loop).
The EVM loop will get the events from the chain, this loop will have events of transaction requests from the EVM compatible chain to Hathor and confirmations on transactions.
The polling loop will check the coordinator service polling api to vote on actions from the MultiSig wallet or check that requests were fulfilled and signal the EVM smart contract.

## Loop descriptions

### EVM loop

We need to get events from the bridge smart contract, wait a pre-configured number of blocks (to increase the confirmation level of the transaction) and send the transaction on Hathor as requested.
There should be 3 types of events:

- A request to cross the token from the EVM chain to Hathor
  - If the token is native to the EVM chain, an equivalent Hathor token should already exist and the MultiSig wallet should own the authority utxos to it.
  - The loop will have to use the coordinator service to send the amount of tokens requested from the MultiSig wallet to the destination address.
- A request to support a new EVM compatible token.
  - We need to check if this is a duplicate request
  - We need to use the coordinator service to create of the equivalent token on Hathor with an amount of 1.00.
  - We need to use the bridge smart contract to save the Hathor equivalent token uid.
- Confirmation that a crossing is over
  - This way we can clean any state pertaining to this crossing of tokens and mark it as complete.
  - If the crossing was originated in Hathor, we need to send a request to the coordinator service so it knows the crossing is over.

### Polling loop

We will pool for events, which can be either a request or a fulfillment.

#### Requests

Requests can be of type "mint", "melt", "unlock", "cross" or "new_token"

- type: _mint request_
  - when: a token native to the EVM chain is trying to cross to Hathor we need to mint the equivalent token to the destination address.
  - it will be a mint transaction on Hathor
- type: _melt request_
  - when: a token native to the EVM chain is trying to cross from Hathor back to the EVM we need to melt the token on Hathor.
  - it will be a melt transaction on Hathor
- type: _cross request_
  - when: a token native to Hathor is trying to cross to the EVM chain we need to vote to create the token on the EVM smart contract.
  - it will not request a transaction on Hathor, only on the EVM chain.
- type: _unlock request_
  - when: a token native to Hathor is trying to cross back to Hathor we need to send the amount requested to the destination address.
  - it will be a simple transaction on Hathor
- type: _new\_token request_
  - when: a user wants the bridge to support a new token native to the EVM chain
  - it will be a create token transaction on Hathor.

#### Fulfillment

Fulfillment events are a way for the coordinator service to inform the federation that a request was fulfilled in Hathor.
This means it has a transaction id sent on Hathor and the request info so the federation can validate the data.

- type: _mint fulfillment_
  - We should inform the smart contract that the operation is finalized.
- type: _unlock fulfillment_
  - We should inform the smart contract that the operation is finalized.
- type: _melt fulfillment_
  - We should unlock the tokens on the EVM chain and send them to the destination address.
- type: _new\_token fulfillment_
  - We should save the token_uid on the EVM smart contract, finally marking the token as supported.
- type: _cross fulfillment_
  - Does not happen since the fulfillment comes from the EVM chain.

## Coordinator service

The coordinator service will serve the function of the smart contract in the EVM side.
It will listen for transactions on the MultiSig wallet, keep a comprehensive persistent database so we do not process the same transaction twice and will gather signatures from the federation.
It will be started with the MultiSig wallet config but it will not be a participant of the wallet, it will use the configuration to derive the MultiSig addresses so it can listen for events on them.

The service will download the history of transactions of the MultiSig wallet and process all transactions.
It will also keep a websocket connection to the fullnode so it can listen for any new transaction in real-time.

### Processing transactions

To process a transaction is to check if it is compliant with the proposal format:

- The first output should be a data output, the data should be the destination address.
- There should only be 1 token being sent to the MultiSig wallet.

If the transaction is compliant, we will save it as a proposal, if not, we will save it as a trash transaction.

- Hathor tx_id
  - With the tx_id, the Hathor loop instance can download the transaction from the fullnode and check that the information is valid.
- Hathor token uid
  - The equivalent EVM token can be found on the smart contract storage (it will be created if this is the first time we see this token)
- amount being sent
- destination address (on EVM chain)

If the token is native to the EVM chain, the coordinator service will also create a "melt request", each service instance will receive the melt request when polling for events.
This request is a transaction hex with the operation to melt the tokens (along with some metadata).
To approve of the melt, the instances will sign the transaction and send the signature to the coordinator service via API.
Once enough signatures are collected, the transaction will be sent on Hathor.

If the token is native to Hathor we will create a "cross request", once it is fulfilled in the smart contract the federation should inform the corrdinator service.

### Actions from the EVM loop

The EVM chain can send events to cross tokens to Hathor or request support for a new token.
These events are "mint request" (for tokens native to the EVM chain), "unlock request" (for tokens native to Hathor) and "new_token request".
These requests will be available via polling api.

The federation will then inspect the request to check the validity then sign and send the signature via API.
Once enough signatures are collected the coordinator service will push the signed transaction and send a fulfillment event.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- We require APIs on the headless wallet to create, mint and melt tokens.
- We require a way to start a MultiSig wallet in the lib without a seed or private key.
  - It is not a read-only since it can send transactions (from gathered signatures)
