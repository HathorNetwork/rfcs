## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
- [Acceptance Criteria](#acceptance_criteria)
- [Overview of Snaps](#overview_of_snaps)
	- [How it works](#overview_of_snaps__how_it_works)
	- [JSON-RPC API](#overview_of_snaps__json_rpc_api)
- [Guide-level explanation](#guide_level_explanation)
	- [UI](#guide_level_explanation__ui)
	- [Configuration](#guide_level_explanation__configuration)
	- [RPC Methods](#guide_level_explanation__rpc_methods)
		- [Request wallet's `xpubkey`](#guide_level_explanation__rpc_methods__request_wallet_xpubkey)
		- [Prove ownership of an address](#guide_level_explanation__rpc_methods__prove_ownership_of_an_address)
		- [Change and verify current network](#guide_level_explanation__rpc_methods__change_and_verify_current_network)
		- [Sign a transaction](#guide_level_explanation__rpc_methods__sign_a_transaction)
- [Reference-level explanation](#reference_level_explanation)
	- [The Snap Manifest](#reference_level_explanation__the_snap_manifest)
	- [The JavaScript Bundle](#reference_level_explanation__the_javascript_bundle)
		- [RPC Methods](#reference_level_explanation__the_javascript_bundle__rpc_methods)
			- [Request Wallet BIP44 `xpubkey`](#reference_level_explanation__the_javascript_bundle__rpc_methods__request_wallet_bip44_xpubkey)
			- [Request Wallet Addresses](#reference_level_explanation__the_javascript_bundle__rpc_methods__request_wallet_addresses)
			- [Prove ownership of an address](#reference_level_explanation__the_javascript_bundle__rpc_methods__prove_ownership_of_an_address)
			- [Change and verify current network](reference_level_explanation__the_javascript_bundle__rpc_methods__change_and_verify_current_network)
			- [Sign a transaction](#reference_level_explanation__the_javascript_bundle__rpc_methods__sign_a_transaction)
- [Conclusion](#conclusion)
	- [Pros](#conclusion_pros)
	- [Cons](#conclusion_cons)
- [Task Break-down](#task_breakdown)

<a name="summary"/>

## Summary

The idea of this document is to give an overview of WalletConnect and evaluate whether it is or not a good option for building our WebWallet and giving support to `dApps` on the Hathor Network

<a name="motivation"/>

## Motivation

Today, web3 is largely web-native. Crypto users are used to working with web wallets, which quickly have become the standard for operations such as login authentication, and performing other actions within DEXes and other `dApps`.

With Nano Contracts approaching, we need to offer our users and builders a way to let our official wallets communicate with web apps.

There will be a single code for integration with Nano Contracts in the wallet lib that any third-party wallets that wish to integrate can use. However, many users and use cases will prefer to work only with our official wallets, so we should enable our mobile and desktop wallets to communicate with the apps being built.

<a name="acceptance_criteria"/>

## Acceptance Criteria

* Evaluation of whether WalletConnect is a good fit with Hathor (i.e. if it supports all operations dapps on hathor will need)
* Evaluate if the technology is ready for being used in production
* Evaluate if the code is well tested and have it's security checked
* Propose a working solution for `dApps` to interact with

<a name="overview_of_wallet_connect"/>

## Overview of WalletConnect

### What is it?

WalletConnect is an open-source protocol that enables secure and seamless communication between decentralized applications (dApps) and wallets on various blockchains. By establishing a remote connection using end-to-end encryption, WalletConnect allows users to interact with dApps through their preferred wallet without exposing their private keys.

### How does it work?

#### 1.  User Flow

User visits a new website that requires a Sign session for submitting transactions and Auth for authentication

1.  `dApp` displays QRCode or deep link
	1. A connection is established with a Relay server and a topic is subscribed
	2. A `session_proposal` message is sent on the channel
2.  User scans QRCode or redirects to wallet (deep-link)
	1. The wallet connects to the same relay server and the same topic is subscribed
1.  Popup with session proposal and authentication request appears on the Wallet
	1. The wallet can either accept or reject the proposal
1.  User approves session request and authenticates
1.  User returns to website after prompt success
1.  User is now authenticated and has a sign session established with the `dApp`

#### 2. Relay Server

The relay server acts as a message broker between the wallet and the dApp. It is responsible for transmitting encrypted messages between the two parties, ensuring that no sensitive information is exposed during communication. The relay server uses the WebSocket protocol to establish real-time connections and does not store any data or have access to the contents of the messages.

The communication process involves the following steps:

**1. Session Request**

The dApp initiates a connection request by generating a QR code containing the connection details. The user then scans this QR code using their wallet, which establishes a WebSocket connection with the relay server.

WalletConnect expects an `Authorization: Bearer <signed jwt>` header when establishing the Websocket connection. Where websocket headers are not supported i.e. browsers, use url query param `?auth=<signed jwt>`. Server should support both of these mechanisms.

The `client_id` is a [DID](https://www.w3.org/TR/did-core/) of the public key for the key pair generated by the client when instantiating the SDK and persisted for the entire lifecycle.

Clients will generate a unique id per app initialization which will be used as a subject when signing a JWT with the `client_id` as the issuer and is persisted through duration of the app lifecycle.

This is the method used to sign the JWT, extracted from the [walletconnect/monorepo](https://github.com/WalletConnect/walletconnect-monorepo/blob/5faa1cc2a8936644f46d946cec08c8af63b082f7/packages/core/src/controllers/crypto.ts#L65) 

```typescript
  public signJWT: ICrypto["signJWT"] = async (aud: string) => {
    this.isInitialized();
    const seed = await this.getClientSeed();
    const keyPair = relayAuth.generateKeyPair(seed);
    const sub = generateRandomBytes32();
    const ttl = CRYPTO_JWT_TTL;
    const jwt = await relayAuth.signJWT(sub, aud, ttl, keyPair);
    return jwt;
  };
```

`this.getClientSeed`  returns a random 32 bytes string that it will either generate or return from storage (it stores after generating)
`relayAuth.generateKeyPair` generates a key pair using `ed25519`
`sub` is the subject, which is an unique id (per client) that will be used when signing the JWT with the `client_id` as the issuer. It's also persisted through the duration of the app lifecycle
`ttl` is a constant, set to one day

The jwt is then signed using the `relayAuth.signJWT` method, this is the implementation:

```typescript
async function signJWT(  
  sub: string,  
  aud: string,  
  ttl: number,  
  keyPair: ed25519.KeyPair  
) {  
  const header = { alg: JWT_ALG, typ: JWT_TYP };  
  const iss = encodeIss(keyPair.publicKey);  
  const iat = fromMiliseconds(Date.now());  
  const exp = iat + ttl;  
  const payload = { iss, sub, aud, iat, exp };  
  const data = encodeData({ header, payload });  
  const signature = ed25519.sign(keyPair.secretKey, data);  
  return encodeJWT({ header, payload, signature });  
}
```

And here is an example JWT:

```javascript
eyJhbGciOiJFZERTQSIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJkaWQ6a2V5Ono2TWtvbndvdXhjVjVjd3hVRUF4UTd3MUxGRzdIcXBxeWFyRkNTRUhtQ0dmYzdhNiIsInN1YiI6Ijk5OTZlNjQ5NDcyZjYxNDEzZjBhNTdiZTY2MzkyZGU5NmY4YzZlM2EwNDI1ZTQwMDE0ZjViNjgzOWNlNmIyZmMiLCJhdWQiOiJ3c3M6Ly9yZWxheS53YWxsZXRjb25uZWN0LmNvbSIsImlhdCI6MTY3OTA2MzA4NCwiZXhwIjoxNjc5MTQ5NDg0fQ.aIuUBxAKgfmq_Vs5BfzEHcmA-PN2IkbJMxS72_pUx2vZ0jylnAV-GNgwzQH0sQOf4a753T_Wofjub47qLccYCQ
```

Which, when decoded, has the following data:

```json
{
  "iss": "did:key:z6MkonwouxcV5cwxUEAxQ7w1LFG7HqpqyarFCSEHmCGfc7a6",
  "sub": "9996e649472f61413f0a57be66392de96f8c6e3a0425e40014f5b6839ce6b2fc",
  "aud": "wss://relay.walletconnect.com",
  "iat": 1679063084,
  "exp": 1679149484
}
```


**2. dApp <-> Wallet linking mechanism**

When the `dApp` gernerates the QRCode for establishing a connection, it generates a topic and subscribes to it, waiting for a `session_proposal` message from the Wallet.

Here is an example URI:

```json
wc:4cfe8a81c7d1c831037a2e766d2797a0d218132cde9b7b2cd3f9b062982e8e38@2?relay-protocol=irn&symKey=b726721e41a339e828597ccf0108cf697b64234c31f61643c148346a0f2a12d5
```

Here is an example subscribe message:

```json
{
  "id": 1679064241504544,
  "jsonrpc": "2.0",
  "method": "irn_subscribe",
  "params": {
    "topic": "4cfe8a81c7d1c831037a2e766d2797a0d218132cde9b7b2cd3f9b062982e8e38"
  }
}
```

After scanning the Wallet parses the URI, extracting the `symKey` which will be used to encrypt messages from now on, the `topic` and the `relay`

It then connects to the relay server (websocket) and subscribes to the topic shared through the `URI`

After the Wallet is also connected to the relay and subscribed to the topic, the `dApp` sends an encrypted `session_proposal` message to the relay server, which then forwards it to the `Wallet`.

Here is an example `session_proposal` message:

```json
{
    "id": 1679064071906409,
    "params": {
        "id": 1679064071906409,
        "pairingTopic": "bb05ad453f1d2cb519ab2b9d69b0776ae4d834fe115cce60a2b1eedaf497b21c",
        "expiry": 1679064408,
        "requiredNamespaces": {
            "hathor": {
                "methods": [
                    "hathor_signMessage"
                ],
                "chains": [
                    "hathor:mainnet"
                ],
                "events": []
            }
        },
        "optionalNamespaces": {},
        "relays": [
            {
                "protocol": "irn"
            }
        ],
        "proposer": {
            "publicKey": "9c4bfcd9c156f23a95ee4a6bd74b68af27f40e17800fd3272d29386e228b9643",
            "metadata": {
                "description": "React App for WalletConnect",
                "url": "http://localhost:3000",
                "icons": [
                    "https://avatars.githubusercontent.com/u/37784886"
                ],
                "name": "React App"
            }
        }
    }
}
```

The Wallet then approves or reject this message, here is an `approve_session` example message, as it is sent from the Wallet to the `dApp` through the relay server, before being encrypted using the `symKey`:

```json
{
    "id": 1679064630962451,
    "relayProtocol": "irn",
    "namespaces": {
        "hathor": {
            "accounts": [
                "hathor:mainnet:HRZZrkp3BcCdvL5RFd4sb3BqXienb5V5yr"
            ],
            "chains": [
                "hathor:mainnet"
            ],
            "events": [],
            "methods": [
                "hathor_signMessage"
            ]
        }
    }
}
```

Once the approve_session message is received and processed by the dApp, the secure communication channel between the dApp and the wallet is established, and they can exchange encrypted messages through the relay server.

Here is a flowchart illustrating this linking process:

```mermaid
graph TB
  dAppGenerateQR[dApp: Generate QR Code] -- URI --> walletScanQR[Wallet: Scan QR Code]
  walletScanQR -- Extract symKey, topic, relay info --> walletConnect[Wallet: Connect to Relay Server]
  walletConnect -- Subscribe to topic --> walletSubscribed[Wallet: Subscribed to Topic]
  dAppConnect[dApp: Connect to Relay Server] --> dAppSubscribed[dApp: Subscribed to Topic]
  dAppSubscribed -- Send encrypted session_proposal --> walletSubscribed
  walletSubscribed -- Receive session_proposal --> walletReview[Wallet: Approve or Reject]
  walletReview -- Send encrypted approve_session --> dAppReceiveApprove[dApp: Receive approve_session]
  dAppReceiveApprove --> dAppSecureCommunication[dApp: Establish secure communication]
  walletReview --> walletSecureCommunication[Wallet: Establish secure communication]
```


**3. Message Exchange** The wallet and the dApp can now exchange encrypted messages through the relay server using the `symKey` shared through the URI (either via QRCode or by Deep-linking).

**4. Session Termination** Either the wallet or the dApp can terminate the session by sending a disconnection message. The relay server then closes the WebSocket connection.

#### 3. Relay Server Concepts

| Field        | Description                                                                                           |
|--------------|-------------------------------------------------------------------------------------------------------|
| Topic        | A string identifier used by clients to publish and subscribe to messages.                             |
| Subscription | Requested by a client and remains active until the client requests to unsubscribe.                     |
| Publication  | Requested by a client; the Relay server is responsible for routing the message to its recipient.       |
| Message      | A utf8 string sent when publishing, to be routed to its recipient.                                    |
| Tag          | An integer used as metadata to identify the type of message being sent.                               |
| TTL          | The maximum amount of time (in seconds) a message should be cached in the relay serverif undelivered.                    |
| Prompt       | A boolean flag used to identify if a message is relevant to registered webhooks.                      |


### Is it secure?

The information sent between `dApps` and wallets is encrypted symmetrically with a key that only these two entities possess

The key is sent directly to the wallet from the `dApp`. Whether users scan through the QR code or choose deep-link with the wallet, there are no intermediaries, so as long as the `symKey` is not leaked, the communication is safe.

This key is generated using the `generateRandomBytes32` method, here is its implementation:

```typescript
export function generateRandomBytes32(): string {
  const random = randomBytes(KEY_LENGTH);
  return toString(random, BASE16);
}
```

`randomBytes` is a method from a library called `stablelib`, which can be seen [here](https://github.com/StableLib/stablelib/blob/a89a438fcbf855de6b2e9faa2630f03c3f3b3a54/packages/random/random.ts#L18).

The implementation for the PRNG used by it can be seen [here](https://github.com/StableLib/stablelib/blob/a89a438fcbf855de6b2e9faa2630f03c3f3b3a54/packages/random/source/browser.ts) for when it's used on a browser, and [here](https://github.com/StableLib/stablelib/blob/a89a438fcbf855de6b2e9faa2630f03c3f3b3a54/packages/random/source/node.ts) when it's used on node.js

### Resources

**Github**: https://github.com/WalletConnect
**Docs**: https://docs.walletconnect.com/2.0/
**Medium**: https://medium.com/walletconnect/
**Sign Reference API**: https://docs.walletconnect.com/2.0/specs/clients/sign/client-api

### SDK

WalletConnect consists of two SDKs available for both the `dApps` and the Wallets: [`Web3Modal`](https://github.com/WalletConnect/web3modal) and [`Web3Wallet`](https://github.com/WalletConnect/walletconnect-monorepo/tree/v2.0/packages/web3wallet):

> Web3Modal is a library that makes it easy to connect users to your `dApp` and interact with the blockchain

It's used to display a HTML popup with the QRCode containing the URI for the peering, it uses the [core](https://github.com/WalletConnect/walletconnect-monorepo/tree/v2.0/packages/core) library to generate the necessary attributes.

> Web3Wallet simplifies the process of integrating wallets into `dApps` by combining the Sign and Auth APIs into a single module

The `web3wallet` library is used on the Wallets for the pairing and message exchange

#### Web3Wallet SDK

> The Web3Wallet SDK allows you to easily connect your Web3 wallet with `dApps`. It provides a simple and intuitive interface for `dApps` to request actions such as signing transactions and interacting with smart contracts on the blockchain. The SDK is designed to work with any blockchain, making it easy to support multiple chains without the need for separate integration code. (https://docs.walletconnect.com/2.0/web3wallet/about)

It's basically a library that provides a single API for the [Sign API](https://docs.walletconnect.com/2.0/api/sign) and the [Auth API](https://docs.walletconnect.com/2.0/api/auth)

#### WalletConnect Cloud

The javascript SDK requires a `projectId` which is supposed to be registered at their WalletConnect cloud, so it can use the WalletConnect infrastructure "for free". 

Note: There is currently no stable open-source implementation of the relay server infrastructure, so it's really necessary.

### Other interesting stuff

* Someone wrote a [kadena implementation proposal](https://github.com/kadena-io/KIPs/blob/jam/wallet-connect/kip-0017.md) to use wallet-connect to communicate between dapps and wallets and also a [new chain request](https://github.com/WalletConnect/walletconnect-monorepo/issues/1905) to wallet-connect


## Guide-level explanation

To go along with this design document, a proof-of-concept has been developed which can be accessed [here](https://github.com/HathorNetwork/hathor-wallet-mobile/pull/245)

The main design decision here is to keep all the business-logic inside the mobile app (or the wallet-desktop), only exposing a JSON-RPC API to the `dApps`, being strict on what methods we want to make available

### UI

There are two parts of the UI when using wallet-connect: The `dApp` UI, which should basically be a QRCode with the information to connect, as described in the "How does it work?" section of this document and that is already provided with the `Web3Modal` library, described in the SDK section of this document.

The other part would be the wallet, which can be either our already existing mobile-wallet or the desktop-wallet, or both.

On the proof-of-concept, I added the WalletConnect to the wallet-mobile project, here is an example of scanning and answering a JSON-RPC request:

<img width="544" alt="Pasted image 20230316114533" src="https://user-images.githubusercontent.com/3586068/225664910-28f3dfba-f508-477f-9950-67ef77dad570.png">
And here is a video demonstrating the pairing mechanism


<video src="https://user-images.githubusercontent.com/3586068/225664948-fd45d6ff-6e1a-4585-99e0-6ea2b914ee54.mov"></video>


## API

For the first implementation of the WebWallet, I suggest we expose the following JSON-RPC APIs


### 1. Prove ownership of an address

`dApps` might ask the wallets to sign an arbitrary message using an address, this method will receive a string and return a string with the signed message encoded in `base64`

### 2. Sign a transaction

The `dApp` should be able to send a transaction to have its inputs validated and signed from the wallets

The wallets should display the same UI from the already existing send transaction screen, but with a rejection button:

<img width="544" alt="Pasted image 20230316114533" src="https://user-images.githubusercontent.com/3586068/225664910-28f3dfba-f508-477f-9950-67ef77dad570.png">

On the mobile wallet, we should also implement a token action confirmation screen, which doesn't yet exist (#todo)


### 3. Get token balance
  
The `htr_getAddressBalance` API provides a way to retrieve the current balance of a specified token


### 4. Get utxos for a transaction

Unlike Ethereum, which uses an account-based model, Hathor employs a UTXO model for managing token balances. In order to create a transaction on a decentralized application (dApp), it is necessary to locate available UTXOs for a specified token. The `htr_getUtxosForToken` API allows developers to find and retrieve UTXOs related to a particular token, facilitating the creation of new transactions.

#todo: other methods are TBD


## Reference-level explanation

In the mobile-wallet, which I used for the proof-of-concept and can be seen [here](https://github.com/HathorNetwork/hathor-wallet-mobile/pull/245/), I used the `@walletconnect/web3wallet` library for all interaction with `wallet-connect`

### Initialization

First, we need to instanciate  `@walletconnect/core`, passing a `PROJECT_ID` that is registered on https://walletconnect.com

This `PROJECT_ID` will be described in-depth in the External Tasks section of this document

Using the `core` instance, we can init the `@walletconnect/web3wallet` instance passing it and some metadata with information about the wallet:

```javascript
  const metadata = {
    name: 'Hathor WalletConnect PoC',
    description: 'Proof-of-concept using WalletConnect to connect to a Hathor dApp',
    url: 'https://hathor.network/',
    icons: ['hathor_logo.png'],
  };
  
  const web3wallet = yield call(Web3Wallet.init, {
    core,
    metadata,
  });
```

After that, we should listen to the `web3wallet` events and react to it, here is an example `setupListeners` method using redux-saga's `eventChannels`:

```javascript
export function* setupListeners(web3wallet) {
  const channel = eventChannel((emitter) => {
    const listener = (state) => emitter(state);
    
    web3wallet.on('session_approval', (proposal) => {
      emitter({
        type: 'WC_SESSION_APPROVAL',
        data: proposal,
      });
    });

    web3wallet.on('session_request', (event) => {
      emitter({
        type: 'WC_SESSION_REQUEST',
        data: event,
      });
    });

    web3wallet.on('session_proposal', async (proposal) => {
      emitter({
        type: 'WC_SESSION_PROPOSAL',
        data: proposal,
      });
    });

    return () => {
      web3wallet.removeListener('session_approval', listener);
      web3wallet.removeListener('session_request', listener);
      web3wallet.removeListener('session_proposal', listener);
    };
  });

  try {
    while (true) {
      const message = yield take(channel);

      yield put({
        type: message.type,
        payload: message.data,
      });
    }
  } finally {
    if (yield cancelled()) {
      channel.close();
    }
  }
}
```

### Pairing

On both wallets, we need to enable the user to scan a QRCode containing the connection URI and also allow him to manage his connected sections

Here is an example `saga` of handling the URI pairing, using the `@walletconnect/core` library:

```javascript
export function* onQrCodeRead(action) {
  const { web3wallet, core } = yield select((state) => state.walletConnect);

  if (!web3wallet) {
    throw new Error('Wallet connect instance is new and QRCode was read');
  }

  const { payload } = action;

  try {
    yield call(() => core.pairing.pair({ uri: payload }));
    console.debug('Pairing...');
  } catch(e) {
    console.error('Error pairing with QrCode: ', e); 
  }
}
```

### Handling session proposals

As described on the "How does it work" section of the WalletConnect overview, a `session_proposal` event is received by the wallet just as soon as the connection is established with the relay server, so we need to handle it by displaying the proposal request to the user with a popup to accept or reject it.

Here is an example saga handling the `onSessionProposal` event:

```javascript
export function* onSessionProposal(action) {
  const { id, params } = action.payload;
  const { web3wallet } = yield select((state) => state.walletConnect);

  const wallet = yield select((state) => state.wallet);
  const addresses = wallet.newAddresses;

  console.log(action.payload);

  const onAcceptAction = { type: 'WALLET_CONNECT_ACCEPT' };
  const onRejectAction = { type: 'WALLET_CONNECT_REJECT' };

  yield put(setWalletConnectModal({
    type: MODAL_TYPES.SESSION_PROPOSAL,
    show: true,
    metadata: {
      title: `${params.proposer.metadata.name} wants to connect.`,
      description: params.proposer.metadata.description,
      requiredMethods: params.requiredNamespaces.hathor.methods,
    },
    onAcceptAction,
    onRejectAction,
  }));

  const { accept } = yield race({
    accept: take(onAcceptAction.type),
    reject: take(onRejectAction.type),
  });

  if (!accept) {
    yield call(() => web3wallet.rejectSession({
      id: params.id,
      reason: {
        code: REJECTION_CODE,
        message: 'User rejected the session',
      },
    }));
  }

  try {
    const sessionApproved = yield call(() => web3wallet.approveSession({
      id,
      relayProtocol: params.relays[0].protocol,
      namespaces: {
        hathor: {
          accounts: accept.payload.selectedAddresses.map((address) => `hathor:mainnet:${address}`),
          chains: [`hathor:${network}`],
          events: [],
          methods: HATHOR_ALLOWED_METHODS,
        },
      },
    }));

    yield call(refreshActiveSessions);
  } catch(e) {
    console.log('Error handling session proposal: ', e);
  }
}
```

In this example, we are displaying a modal describing the methods that the `dApp` is requesting and after approal, it sends an `session_approved` method through the `web3wallet` instance.

The modal must display a list of the wallet's addresses and allow the user to select one or more of them to connect (see the design decisions section)

### Handling session requests

After the session is established, we will receive RPC requests through the `session_request` event, here is an example request:

```json
{
    "id": 1679071024881602,
    "topic": "d5b50afb3541aea8ce9c8a0341a18bff913d3d83602084424cffcd3b81c28626",
    "params": {
        "request": {
            "method": "hathor_signMessage",
            "params": {
                "address": "HRZZrkp3BcCdvL5RFd4sb3BqXienb5V5yr",
                "message": "Please sign me!"
            }
        },
        "chainId": "hathor:mainnet"
    }
}
```

As we can see, in the params we can see the requested method and the params, in this case, the address to sign with and the message for the `signMessage` RPC API

Handling this specific message will be described in the API section of the reference-level explanation
