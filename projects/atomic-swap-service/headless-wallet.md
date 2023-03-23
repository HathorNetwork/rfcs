# Summary
[summary]: #summary

This design describes the integration of the [Headless Wallet](https://github.com/HathorNetwork/hathor-wallet-headless) with the [Atomic Swap Service](https://github.com/HathorNetwork/hathor-atomic-swap-service) backend.

This will allow for easy swap operations between desktop and headless clients, making communication easier for automated operations, such as websites facilitating swaps for end-users.

# Acceptance criteria
[acceptance-criteria]: #acceptance-criteria

- Any initialized wallet on the headless app should be able to participate on atomic swap proposals managed by the Atomic Swap Service
- Each wallet's end users should be able to register proposals to be listened to, and receive real-time change notifications for them
- There should be no impact on the current atomic swap workflow based on string exchanges

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The headless wallet, in a similar fashion as [what is proposed on the Desktop Wallet](https://github.com/HathorNetwork/hathor-wallet/pull/361/files#diff-81b333677b8eb7fcc977d225072c1c10453c0aee095f9458885d7a553ef3579d), will have a `listnenedProposals` map in memory for each of the initialized wallets and will keep track of their identifiers, passwords and real-time updates.

### Example use case 1
A user for the initialized wallet `swap-website-wallet` calls the `registerProposal` endpoint with an identifier `prop-1` and password `123` that it has received from another one of the proposal's participant. This request will add `prop-1` to the local `listenedProposals` object in memory.

A `fetch` call can be made right after, informing the desired `prop-1` identifier, to retrieve its latest state on the service backend and, after changes are made,  `update` requests can persist the changes and inform the other swap participants about it in real time.

### Example use case 2
A user of the initialized wallet `swap-website-wallet` calls the `atomic-swap/tx-proposal` passing to it, besides all the necessary parameters to initialize a proposal, the new parameters `service.is_new` and `service.password`. This will make the headless app also initialize this proposal on the service and add it to its local `listenedProposals`.

These identifier and password can then be shared with the other interested participants and future interactions can follow just like on example 1.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Listened proposals
This integration with the swap service backend needs a way for the headless to store in memory the proposals that are currently being monitored by each of its initialized wallets.

There are some considerations about this object:
- An event listener will be created for each wallet to receive web-socket updates from the service
- More than one initialized wallet can be listening to the same proposal
- Wallets that did not register a proposal should not receive updates about it

To achieve this, a `walletListenedProposals` map will be created on the `services/wallets.service` module.
```js
/**
 * @typedef TxProposalConfig
 * @property {string} id Proposal identifier on the Atomic Swap Service
 * @property {string} password Password to access it
 */

/**
 * A map of all the proposals for a wallet.
 * The keys are the proposal ids
 * @typedef {Record<string,TxProposalConfig>} WalletListenedProposals
 */

/**
 * A map of the initialized wallets and their listened proposals.
 * The keys are the wallet-ids
 * @type {Record<string,WalletListenedProposals>}
 */
const walletListenedProposals = {};
```

For simplicity, references to `listenedProposals` on this document refer to the listened proposals for a specific initialized wallet

### Create endpoint
The `[POST] /wallet/atomic-swap/tx-proposal` route will receive the new optional parameter `service`:
```ts
{
	is_new: boolean,
	password: string,
}
```
Upon a successful creation, a new property `createdProposalId` will be available on the response object, and its data will be registered on the `listenedProposals` accordingly.

### Register endpoint
A new route `[POST] /wallet/atomic-swap/tx-proposal/register/${proposalId}` will be created, receiving as body parameters:
```ts
{
	password: string,
}
```
This will allow the wallet user to add any existing proposal to the wallet's `listenedProposals` and interact with them.

To ensure the informed parameters are correct, a `get` request will also be made at this time to the service backend, and a validation executed on the received contents. Any failure found on the process will be thrown and the registration discarded.

### Listened Proposals endpoint
A new route `[GET] /wallet/atomic-swap/tx-proposal/list` will be created, allowing the user to retrieve the list of `proposalIds` currently being listened to. No password should be retrieved on this call.

### Fetch endpoint
A new route `[GET] /wallet/atomic-swap/tx-proposal/fetch/{proposalId}` will be created, allowing the user to pool the service backend for updates on a registered proposal.

Calls to this route for a `proposalId` that has not been registered will raise a `404` error.

Should the backend return a `404` error, the proposal is considered obsolete and can be removed from the `listenedProposals` map.

### Update endpoint
The [current workflow](https://hathor.gitbook.io/hathor/guides/headless-wallet/atomic-swap#step-3-bob-updates-alices-partial-transaction) of updating proposals will be kept unaltered, requiring calls to the `[POST] /wallet/atomic-swap/tx-proposal` route. However, in order to persist the changes on the Atomic Swap Service, the additional body parameter `service.proposal_id` must also be informed on the request.

Calling this route with a proposal identifier that has not been registered will raise a `400` error. Any errors raised while interacting with the service will also be treated and returned on the http response.

Should the backend return a `404` error, the proposal is considered obsolete and can be removed from the `listenedProposals` map.

### Unregister endpoint
A new route `[DELETE] /wallet/atomic-swap/tx-proposal/delete/{proposalId}` will be created, allowing the user to stop listening to this proposal for the wallet specified on the request header.

Also, should a wallet be stopped, all proposals it's currently listening to should also be removed, and its websocket connection closed.

### External notifications
For every wallet with a registered proposal, a websocket connection will be opened with the Atomic Swap Service, informing the application about any changes that happen to its proposals.
The event emitted by the lib will be `update-atomic-swap-proposal`, keeping consistent with the [current event names](https://github.com/HathorNetwork/hathor-wallet-headless/blob/4fb465eb8420ea93dbcd43a6c091453b74dbfded/src/services/notification.service.js#L30).

Whenever a message arrives through this lib channel, the corresponding wallet user will receive a notification through the _External Notifications_ feature containing the updated proposal full contents. The external notification event name will be `wallet:update-atomic-swap-proposal`, and will be added to the [`WalletEventMap` property](https://github.com/HathorNetwork/hathor-wallet-headless/blob/4fb465eb8420ea93dbcd43a6c091453b74dbfded/src/services/notification.service.js#L14) on the notification service.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Lean proposal objects
In contrast to the desktop wallet, that keeps all the proposal data in memory, the headless keeps only the identifier, password and listeners.

The current atomic swap headless workflow already expects the application not to keep any of the proposal data cached, so this would not present a difference in its usability.

The alternative of keeping all the proposal's serialized `PartialTx` and `signatures` would not add any relevant ease of use while increasing the application's memory consumption.

# Future possibilities
[future-possibilities]: #future-possibilities

### Automatic discard of completed atomic swaps
A feature can be implemented to identify when a fully signed atomic swap proposal mediated by the Atomic Swap Service has been turned into a transaction.

This would involve adding functionality to the `services/notification.service` module, building on [the existing hook](https://github.com/HathorNetwork/hathor-wallet-headless/blob/4fb465eb8420ea93dbcd43a6c091453b74dbfded/src/services/notification.service.js#L63-L69) for the *HathorWallet* `new-tx` event. If all inputs and outputs are identical to one of the proposals being listened to ( independently of the initialized wallet ), it should be safe to assume the proposal can be removed from the `listenedProposals` map.
