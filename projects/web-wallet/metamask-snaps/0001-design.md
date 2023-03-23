* Feature name: Using MetaMask Snaps for our WebWallet
* Start date: 27/02/2023
* [Initial Document](https://docs.google.com/document/d/1gJ44kSN3o1LXGOlfsVUTggqD7wV84FDIJ3Nkicx0zcc/edit)
* Author: @andreabadesso

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

The idea of this document is to give an overview of MetaMask Snaps and evaluate whether it is or not a good option for building our WebWallet and giving support to `dApps` on the Hathor Network.

## Motivation

Today, web3 is largely web-native. Crypto users are used to working with web wallets, which quickly have become the standard for operations such as login authentication, and performing other actions within DEXes and other `dApps`.

With Nano Contracts approaching, we need to offer our users and builders a way to let our official wallets communicate with web apps.

There will be a single code for integration with Nano Contracts in the wallet lib that any third-party wallets that wish to integrate can use. However, many users and use cases will prefer to work only with our official wallets, so we should enable our mobile and desktop wallets to communicate with the apps being built.

<a name="motivation2"/>

## Acceptance Criteria

* Evaluation of whether Snaps are a good fit with Hathor (i.e. if it supports all operations dapps on hathor will need)
* Evaluate if the technology is ready for being used in production
* Evaluate if the code is well tested and have it's security checked
* Propose a working solution for `dApps` to interact with

<a name="overview_of_snaps"/>

## Overview of Snaps

Snaps is a system that allows anyone to safely extend the capabilities of MetaMask. A _snap_ is a program that is run in an isolated environment inside MetaMask and that can customize the wallet experience.

For example, a snap can add new APIs to MetaMask, add support for different blockchain protocols, or modify existing functionality using internal APIs. Snaps is a new way to create web3 end user experiences, by modifying MetaMask in ways that were impossible before.

Currently, Snaps are released as pre-release software and only runs in what is called the [Flask version](https://metamask.io/flask/) of the Metamask wallet, which is described as a "experimental playground for developers, where new or proposed features can be rolled out and tested before they are deployed (...)".

<a name="overview_of_snaps__how_it_works"/>

### How it works

Snaps are untrusted JavaScript programs that execute safely inside the MetaMask application. To isolate snaps from the rest of the applications and to provide a "fully virtualizable" execution environment, MetaMask uses [Secure ECMAScript (SES)](https://github.com/endojs/endo/tree/master/packages/ses), a subset of Javascript developed by [Agoric](https://agoric.com/).

More on this can be read [here](https://github.com/tc39/proposal-ses).

<a name="overview_of_snaps__json_rpc_api"/>

### JSON-RPC API

Besides running on a secure and limited environment, Snaps have access to a limited set of capabilities, determined by the permissions they were granted by the user during installation. As with MetaMask’s [Ethereum Provider RPC API](https://docs.metamask.io/guide/rpc-api.html) available for dApps, snaps also communicate with MetaMask using JSON-RPC.

There are many available features for the snaps using the RPC methods, I selected the most interesting ones for building our WebWallet (the complete list can be read [here](https://docs.metamask.io/guide/snaps.html#features)):

1. **Display a custom confirmation screen in MetaMask**

Show a MetaMask popup with custom text and buttons to approve or reject an action. This can be used to create requests, confirmations, and opt-in flows for a snap, on Hathor, ideally we should ask for user confirmation on all actions that involve his private or public keys, this method allows for exactly this.

2. **Notify users in MetaMask**

MetaMask Flask introduces a generic notifications interface that can be utilized by any snap with the notifications permission. A short notification text can be triggered by a snap for actionable or time-sensitive information.

In practice, we can display notifications in the browser (if the user allows it) at any point from the hathor snap

3. **Store and manage data on your device**
   
Store, update, and retrieve data securely, with encryption by default until a limit of 100MB of data.

4. **Control non-EVM accounts and assets in MetaMask**

Derive BIP-32 and BIP-44 keypairs based on the Secret Recovery Phrase without exposing it. With the power to manage keys, you can build snaps to support a variety of blockchain protocols.

5. **Custom UI in MetaMask using a defined set of components**
   
Display Custom UI within MetaMask using a set of pre-defined components, including inline Markdown.

This might be used to sign special transactions from dapps, like nanocontract transactions or even display a received or sent NFT image on the transaction confirmation.

<a name="guide_level_explanation"/>

## Guide-level explanation

To go along with this design document, a proof-of-concept has been developed which can be accessed [here](https://github.com/hathornetwork/htrsnap/tree/dev) — I wasn't able to host it because MetaMask won't allow us to install without publishing it on npm

The main design decision here is that the `dApp` will receive a `xpubkey` and will be responsible for connecting either with a `fullnode` or with the `WalletService`, only interacting with the Snap RPC API when doing an action that requires access to one of the wallet's private keys.

Refer to the [[#Guide-level explanation#RPC Methods|RPC Methods]] section of the guide-level explanation for more details on the RPC API.

<a name="guide_level_explanation__ui"/>

### UI 

My suggestion is to port the `hathor-wallet-desktop` project to work on the web and add support on the Wallet Facades to communicate with the new `HathorSnap`. This way, we can keep a single code base for our desktop and browser apps.

<a name="guide_level_explanation__configuration"/>

### Configuration

The Snap provides a RPC method to connect the dApp with the Snap, this is handled by MetaMask and needs no action from the Snap itself. 

The user is greeted with this message displaying the website that requested to connect and the name of the Snap it wants to connect to.

<p align="center">
  <img src="https://user-images.githubusercontent.com/3586068/223004603-ae499a68-4d83-4d33-81c1-5051f2786ec8.png" />
</p>

When the snap is deployed, it will display the name of the snap (e.g. `htrsnap`) instead of `local:http://localhost:8081/Snap`.

After the user accepts the connection request, the snap will display the necessary permissions it requires, again for user validation:

<p align="center">
  <img src="https://user-images.githubusercontent.com/3586068/223004695-20f33bf7-40b0-47f3-a3ea-16200d76a81c.png" />
</p>

And it makes sure the user understands what he's giving permission:
<p align="center">
  <img src="https://user-images.githubusercontent.com/3586068/223004780-1e983da4-bec0-423a-903e-6dd4872268ab.png" />
</p>
**Note:** After the permissions have been granted, the MetaMask wallet will only display hathor-related information when requested, the wallet that appears loaded in the MetaMask extension is the one managed by MetaMask when the wallet was created/loaded:
<p align="center">
  <img src="https://user-images.githubusercontent.com/3586068/223004857-ea0d6acd-dc67-4ac4-a061-29190f28332f.png" />
</p>

<a name="guide_level_explanation__rpc_methods"/>

### RPC Methods

The idea is to develop RPC methods that will allow the following actions from `dApps`:

<a name="guide_level_explanation__rpc_methods__request_wallet_xpubkey"/>

#### Request wallet's `xpubkey`

dApps might request the current loaded wallet's `xpubkey`, this should open a permission dialog so that the user knows which website requested his `xpubkey` and allow or disallow it.

The idea is that dApps will never have access to the wallet's private keys directly, only through RPC calls that will always ask for user permission with a descriptive message.

<a name="guide_level_explanation__rpc_methods__prove_ownership_of_an_address"/>

#### Prove ownership of an address

`dApps` might also ask the Snap to sign an arbitrary message using an address, this method will receive a string and return a string with the signed message encoded in `base64`, proving that the user has access to the private key of that address

<p align="center">
  <img src="https://user-images.githubusercontent.com/3586068/223004960-1992de1b-9bb9-41ae-8548-5b5dcf1974dc.png" />
</p>

<a name="guide_level_explanation__rpc_methods__change_and_verify_curr_network"/>

#### Change and verify current network

Since there is no available UI on MetaMask to manually change the network, the Snap should be able to manage it's network and we should provide `dApps` the RPC calls to change and verify the currently configured network. We should store this setting on the MetaMask storage, which is also described in this document.

<a name="guide_level_explanation__rpc_methods__sign_a_transaction"/>

#### Sign a transaction

The `dApp` should be able to send a transaction to have its inputs validated and signed with the `privatekey` stored in MetaMask.

The Snap should display a user friendly interface with all the inputs and outputs involved in the transaction so the user can double check if that's exactly what he intends to do.
<p align="center">
  <img src="https://user-images.githubusercontent.com/3586068/223005440-a0b48393-27c3-4bb2-a442-4be0ca0c6066.png" />
</p>


These methods were created considering a functional wallet started from the `xpubkey` and using the wallet-lib, more RPC methods might be added if needed

<a name="reference_level_explanation"/>

## Reference-level explanation

A Snap consists of two things: A JSON manifest and a JavaScript bundle. At present, Snaps can be published as npm packages on the public npm registry or locally during development.

If we decide with building our Snap, we will publish it on the public npm registry.

<a name="reference_level_explanation__the_snap_manifest"/>

### The Snap manifest

This file will describe the Snap and declare the required permissions, for the first version of the HathorSnap, we will declare these permissions:

```json
  "initialPermissions": {
    "endowment:rpc": {
      "dapps": true,
      "snaps": false
    },
    "snap_confirm": {},
    "snap_dialog": {},
    "snap_manageState": {},
    "snap_getBip44Entropy": [
      {
        "coinType": 280
      }
    ]
  },
```

* `endowment:rpc` -- This permission is required for snaps that need to handle arbitrary JSON-RPC requests, this permission grants a snap access to JSON-RPC requests sent to it, using the `onRpcRequest` method. We should only receive rpc calls from `dApps` for now, but we might allow RPC requests from other snaps in the future, this would allow the community to extend our Snap funcionality.
* `snap_confirm` -- This RPC method causes a confirmation to be displayed in the MetaMask UI. This is used to display Hathor-related information on the extension's UI.
* `snap_dialog` -- Similar to the `snap_confirm`, this is used to display arbitrary information on the MetaMask UI.
* `snap_manageState` -- This RPC method allows the Snap to store arbitrary information to disk. This data is automatically encrypted using a snap-specific key and automatically decrypted when retrieved.
* `snap_getBip44Entropy` -- This RPC method allows the Snap to get the BIP-44 key for the `coin_type` number specified by the method name. This allows the snap to derive private keys for all addresses on the loaded wallet.

<a name="reference_level_explanation__the_javascript_bundle"/>

### The JavaScript Bundle

The JavaScript bundle should basically export an `onRpcRequest` method that will be called with the RPC requests sent by the Snap through the `wallet_invokeSnap` RPC API

The interface for invoking a snap RPC method is:

```typescript
interface SnapRequest {
  method: string;
  params?: unknown[] | Record<string, unknown>;
  id?: string | number;
  jsonrpc?: '2.0';
}
```

Here is an example on the client for requesting wallet addresses:

```javascript
async function getHathorAddresses() {
  const response = await ethereum.request({
	method: 'wallet_invokeSnap',
	params: {
	  snapId,
	  request: {
		method: 'htr_getaddresses',
		params: {
		  from: 0,
		  to: 19,
		},
	  }
	}
  });
  ...
}
```

And the definition for the `onRpcRequest` method that will handle it:

```typescript
export const onRpcRequest = async ({origin, request}: RpcRequest) => {
  switch (request.method) {
    case 'htr_getaddresses':
      return getHathorAddresses(
        origin,
        snap,
        request.params.from,
        request.params.to,
      );
	...
  }
}
```

The implementation for `getHathorAddresses` will be described on the `RPC Methods` section of this reference-level explanation

<a name="reference_level_explanation__the_javascript_bundle__rpc_methods"/>

#### RPC Methods
<a name="reference_level_explanation__the_javascript_bundle__rpc_methods__request_wallet_bip44_xpubkey"/>

##### Request Wallet BIP44 `xpubkey`

**Method**: `htr_getxpubkey`
**Params**: -
**Returns**: A string containing a wallet's `xpubkey` at the `m/44'/280'` derivation path
**Example implementation**:

```typescript
export async function getHathorRootNode(snap: Snap): Promise<BIP44CoinTypeNode> {
  const hathorNode = await snap.request({
    method: 'snap_getBip44Entropy',
    params: {
      coinType: parseInt(HATHOR_BIP44_CODE, 10),
    },
  }) as BIP44CoinTypeNode;

  return hathorNode;
}

export async function getHathorXPubKey(origin: string, snap: Snap): Promise<{ xpub: string }> {
  await snap.request({
    method: 'snap_dialog',
    params: {
      type: 'Confirmation',
      content: panel([
        heading('Access your extended public key'),
        text(`${origin} is trying to access your Hathor extended public key.`),
      ]),
    },
  });

  const hathorNode = await getHathorRootNode(snap);

  const privateKeyBuffer = Buffer.from(trimHexPrefix(hathorNode.privateKey), 'hex')
  const chainCodeBuffer = Buffer.from(trimHexPrefix(hathorNode.chainCode), 'hex')
  const node: BIP32Interface = bip32.fromPrivateKey(privateKeyBuffer, chainCodeBuffer, hathorNetwork)
  const xpub = xpubFromNode(node);

  return { xpub };
}
```

<a name="reference_level_explanation__the_javascript_bundle__rpc_methods__prove_ownership_of_an_address"/>

##### Prove ownership of an address

**Method**: `htr_proveaddress`
**Params**: `{ addressIndex: number, message: string }`
**Returns**: A base64-encoded string with the signed message
**Example implementation**:
```typescript
export async function signHathorMessage(
  origin: string,
  snap: Snap,
  message: string,
  addressIndex: number,
): Promise<{ signedMessage: string }> {
  const hathorNode = await getHathorRootNode(snap);
  const addressNode: BIP44Node = await getAddressAtIndex(hathorNode, addressIndex);
  const address: string = addressNodeToP2PKHAddress(addressNode, hathorNode);

  await snap.request({
    method: 'snap_dialog',
    params: {
      type: 'Confirmation',
      content: panel([
        heading('Sign message'),
        text(`${origin} is trying to sign the following message:`),
        text(message),
        text(`With the following address: ${address}`),
      ]),
    },
  });

  const privateKeyBuffer = Buffer.from(trimHexPrefix(addressNode.privateKey), 'hex')

  const signature = bitcoinMessage
    .sign(
      message,
      privateKeyBuffer,
      true,
      hathorNetwork.messagePrefix,
    )
    .toString('base64');

  return {
    signedMessage: signature, // signature.toString('base64'),
  };
}
```

<a name="reference_level_explanation__the_javascript_bundle__rpc_methods__request_wallet_addresses"/>

##### Request Wallet Addresses

**Method**: `htr_getaddresses`
**Params**: `{ from: number, to: number }`
**Returns**:
```typescript
interface Address {
  index: number;
  path: string;
  base58: string;
}

type GetAddressesResponse = Address[];
```

**Example implementation**:

```typescript
export const hathorNetwork = {
  messagePrefix: '\x18Hathor Signed Message:\n',
  bech32: 'ht',
  bip32: {
    public: 76067358,
    private: 55720709,
  },
  pubKeyHash: 40,
  scriptHash: 100,
  wif: 128
};

export const getAddressAtIndex = async (hathorNode: BIP44CoinTypeNode, index: number): Promise<string> => {
  const hathorAddressDeriver = await getBIP44AddressKeyDeriver(hathorNode);
  const derivedAddress = await hathorAddressDeriver(index);

  const privateKeyBuffer = Buffer.from(trimHexPrefix(derivedAddress.privateKey), 'hex')
  const chainCodeBuffer = Buffer.from(trimHexPrefix(hathorNode.chainCode), 'hex')
  const node: BIP32Interface = bip32.fromPrivateKey(privateKeyBuffer, chainCodeBuffer, hathorNetwork)

  return bitcoin.payments.p2pkh({
    pubkey: node.publicKey,
    network: hathorNetwork,
  }).address;
};

export async function getHathorAddresses(_origin: string, snap: Snap, from: number, to: number): Promise<{ addresses: string[] }> {
  const hathorNode = await getHathorRootNode(snap);

  const addresses: string[] = [];
  for (let index = from; index <= to; index++) {
    addresses.push(await getAddressAtIndex(hathorNode, index));
  }

  return { addresses };
}
```

<a name="reference_level_explanation__the_javascript_bundle__rpc_methods__change_and_verify_current_network"/>


##### Change and verify current network

**Method**: `htr_getnetwork`
**Params**:
**Returns**:
```typescript
interface GetNetworkResponse {
  network: 'mainnet' | 'testnet' | 'privatenet';
  websocketUrl: string;
  httpUrl: string;
}
```

**Method**: `htr_setnetwork`
**Params**:
```typescript
interface SetNetworkParams{
  websocketUrl: string;
  httpUrl: string;
}
```

**Returns**:
```typescript
type SetNetworkResponse = GetNetworkResponse;
```

<a name="reference_level_explanation__the_javascript_bundle__rpc_methods__request_wallet_addresses"/>

##### Sign a transaction

**Method**: `htr_signtransaction`
**Params**:
```typescript
interface Input {
  txId: string;
  index: number;
}

interface Output {
  type: string;
  value: number;
  token: string;
  address: string; // Required because the dApp needs to do all the work
  timelock?: number | null;
  data?: string; // Required for data script
}

interface SignTransactionParams {
  inputs: Input[];
  outputs: Output[];
}
```

**Returns**:
```typescript
interface GetAddressesResponse {
  txHex: string;
}
```

<a name="conclusion"/>

## Conclusion

The Snap feature on its current version already has support for all features we need to allow `dApps` to communicate securely with the Hathor Network, the documentation is great and clear and with multiple examples. There are also multiple projects already using Snap, like the `btcsnap` and others.

Also, MetaMask provides us with already validated security features that were built during the last 7 years they exist with over 20 million users using it daily with real money, that fact can not be understated.

But even though it looks very promising, it is not yet in production and the expected ETA for the production version is at least in Q3 2023. It also did not go through a complete security audit yet (I've been informed that it's in progress) and thus needs more validation before we can use it in production.

If we decide to have a working web wallet that is ready to be released in production before this date, we should go for other solutions.

<a name="conclusion_pros"/>

### Pros

- [ ] Creating a Snap does not entail any explicit costs, it is open-source software and should be free to use for non-commercial purposes
- [ ] Documentation is [widely available](https://docs.metamask.io/guide/snaps.html) with examples and helper libraries
- [ ] MetaMask is a widely used wallet in the web3 space. Even though it might be confusing at first to not use the MetaMask UI to interact with Hathor `dApps`, users can use their existing eth/bsc/... wallets to interact with Hathor, which is something that is very common in this space.
- [ ] MetaMask has an [active bug bounty](https://hackerone.com/metamask) paying up to $50k for a critical vulnerability
- [ ] There are many interesting applications already using Snaps, notably the [btcsnap](https://github.com/snapdao/btcsnap), which is similar to what we are planning to do.
- [ ] With MetaMask, we can benefit from many years of experience the MetaMask team has on safely storing private keys on browsers and also from regular updates as there is a full-time team working on it

<a name="conclusion_cons"/>

### Cons
- [ ] No support for ledger (yet), even though MetaMask supports it. I asked on the ConsenSys discord and they have plans to provide a USB API, but it shouldn't be expected anytime soon
- [ ] Snaps is only supported on the Flask version (canary) of metamask and the ETA for the initial rollout is at least Q3 2023
- [ ] The Snaps feature is still undergoing a security audit, with an ETA of at least Q3 2023
- [ ] The MetaMask mobile app has no support for Snaps

<a name="task_breakdown"/>

## Task break-down

| Task | Dev/days |
| --- | --- |
| Implement the RPC methods described in the Reference-level explanation | 8 |
| Transform the `hathor-wallet-desktop` project to run in the web and use the HathorSnap | 4 |
| Adapt the wallet-lib to run in SES | 1 |
| Infrastructure for deploying the new wallet | 2 |
| QA and internal testing | 2 |

Total: 17 dev/days

---
Reviewers:
@pedroferreira, @msbrogli, @trondbjoroy
