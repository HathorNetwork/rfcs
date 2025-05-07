# Summary

The goal of this document is to describe/discuss how to implement a snaps, which features it will have, the process to go live with it and the need for a web wallet.

# Motivation

The motivation and snaps overview sections have been copied from [a previous design](https://github.com/HathorNetwork/rfcs/blob/ce630677981eebdffb7efd0af623e5f0fc1c51df/projects/web-wallet/metamask-snaps/0001-design.md) that already described in great details only with some small changes and updates from the latest snaps versions.

Today, web3 is largely web-native. Crypto users are used to working with web wallets, which quickly have become the standard for operations such as login authentication, and performing other actions within DEXes and other `dApps`.

With Nano Contracts approaching, we need to offer our users and builders a way to let our official wallets communicate with web apps.

There will be a single code for integration with Nano Contracts in the wallet lib that any third-party wallets that wish to integrate can use. However, many users and use cases will prefer to work only with our official wallets, so we should enable our mobile and desktop wallets to communicate with the apps being built.

# Overview of Snaps

Snaps is a system that allows anyone to safely extend the capabilities of MetaMask. A _snap_ is a program that is run in an isolated environment inside MetaMask and that can customize the wallet experience.

For example, a snap can add new APIs to MetaMask, add support for different blockchain protocols, or modify existing functionality using internal APIs. Snaps is a new way to create web3 end user experiences, by modifying MetaMask in ways that were impossible before.

The only way to install the development snaps is to use what is called the [Flask version](https://metamask.io/flask/) of the MetaMask wallet, which is described as a "experimental playground for developers, where new or proposed features can be rolled out and tested before they are deployed (...)".

After the code is implemented, tested and ready to be published, it must go through the [publishing process](https://docs.metamask.io/snaps/how-to/publish-a-snap/) and then through [auditing and allowlist process](https://docs.metamask.io/snaps/how-to/publish-a-snap/).

### How it works

Snaps are untrusted JavaScript programs that execute safely inside the MetaMask application. To isolate snaps from the rest of the applications and to provide a "fully virtualizable" execution environment, MetaMask uses [Secure ECMAScript (SES)](https://github.com/endojs/endo/tree/master/packages/ses), a subset of Javascript developed by [Agoric](https://agoric.com/).

More on this can be read [here](https://github.com/tc39/proposal-ses).

### JSON-RPC API

Besides running on a secure and limited environment, Snaps have access to a limited set of capabilities, determined by the permissions they were granted by the user during installation. As with MetaMask’s [Ethereum Provider RPC API](https://docs.metamask.io/guide/rpc-api.html) available for dApps, snaps also communicate with MetaMask using JSON-RPC.

There are many available features for the snaps using the RPC methods, I selected the most interesting ones for building our WebWallet (the complete list can be read [here](https://docs.metamask.io/guide/snaps.html#features)):

1. **Display a custom confirmation screen in MetaMask**

Show a MetaMask popup with custom text and buttons to approve or reject an action. This can be used to create requests, confirmations, and opt-in flows for a snap, on Hathor, ideally we should ask for user confirmation on all actions that involve his private or public keys, this method allows for exactly this. [See it in details](https://docs.metamask.io/snaps/features/custom-ui/dialogs/#display-a-confirmation-dialog).

2. **Notify users in MetaMask**

MetaMask Flask introduces a generic notifications interface that can be utilized by any snap with the notifications permission. A short notification text can be triggered by a snap for actionable or time-sensitive information.

In practice, we can display notifications in the browser (if the user allows it) at any point from the hathor snap.

This could be useful for notifying the user when a transaction has been confirmed by a block. Nowadays nano contract methods are executed once a block confirms the transaction, so the user can know when the method has been executed and check if it had any errors. [See it in details](https://docs.metamask.io/snaps/features/notifications/)

3. **Store and manage data on your device**

Store, update, and retrieve data securely, with encryption by default until a limit of 100MB of data. [See it in details](https://docs.metamask.io/snaps/features/data-storage/)

4. **Control non-EVM accounts and assets in MetaMask**

Derive BIP-32 and BIP-44 keypairs based on the Secret Recovery Phrase without exposing it. With the power to manage keys, you can build snaps to support a variety of blockchain protocols. [See it in details](https://docs.metamask.io/snaps/features/non-evm-networks/)

5. **Custom UI in MetaMask using a defined set of components**

Display Custom UI within MetaMask using a set of pre-defined components, including inline Markdown.

This might be used to sign special transactions from dapps, like nanocontract transactions or even display a received or sent NFT image on the transaction confirmation. [See it in details](https://docs.metamask.io/snaps/features/custom-ui/)

# Guide-level Explanation

## Snaps

The MetaMask snaps works similar to the Wallet Connect, where dApps can communicate through RPC calls to sign transactions, get addresses, get balance, among other actions.

The MetaMask extension has its own root private key and we can derive private keys in the Hathor Network derivation path. The main benefit of this is that we don't store any sensitive information from the user, we just use the MetaMask tools to derive our addresses.

### Running a wallet

Although it is possible to store data among RPC calls, real time update is not possible using snaps anymore (they removed support for [`endowment:long-running`](https://github.com/MetaMask/snaps/pull/1126) and [`websocket`](https://github.com/MetaMask/snaps/pull/1122/files)).

Because of that, the wallet service is the best choice for starting a wallet every time a RPC call is requested. Despite starting a websocket object, it's just for real time update and it's easy enough to implement a patch for removing this requirement.

#### Using wallet service

The wallet service is still not ready to be used by the MetaMask snaps, especially because it currently still doesn't support nano contracts and its wallet lib in use is old.

##### Wallet lib upgrade

Wallet service is using wallet lib `v0.39.0`, and our latest wallet lib version is `v2.0.1`. We released 37 versions between them, with 2 major version changes. The approach for this upgrade will be to first upgrade to `v1.14.1`, which has the new storage scheme but without the big int support. Then, we will add this big int support, upgrading to `v2.0.1`. This last upgrade will need lot of changes in the code itself to support the big int values.

[UPDATE] The wallet service is already using wallet-lib `v1.14.0` and the task to upgrade to `v2.1.1` is in progress.

##### Support for nano contracts

This will be done by @andreabadesso in a different project, and its design is already in progress.

##### Support all features

We don't have all facade methods from the `HathorWallet` class implemented in the `HathorWalletServiceWallet` class. We must have full support, in order to be able to implement the RPC methods.

The least we need to have is:

1. `createAndSendNanoContractTransaction` and `createNanoContractTransaction` for nano contract support.
1. `isAddressMine` to use in nano contract validations.
1. `getAddressInfo` to use in the `getAddress` RPC method.
1. `getTx` and `getFullHistory` to use in some RPC methods.


NOTE: We currently don't support multisig in the wallet service facade but I think it's okay to continue with this for now.
 
### Implementation Phases

#### RPC methods

The first version of the Hathor Snaps will have support for all RPC methods we currently support in our [RPC library](https://github.com/HathorNetwork/hathor-rpc-lib).

The methods we will have support are:

1. `createToken`
1. `getAddress`
1. `getBalance`
1. `getConnectedNetwork`
1. `getUtxos`
1. `sendNanoContractTx`
1. `signOracleData`
1. `signWithAddress`
1. `sendTransaction`

## Web Wallet

A web wallet is not a requirement for having a dApp integrated with our snaps, it's possible to simply call RPC methods and interact directly with MetaMask only using the dApp, without a wallet. However, MetaMask won't show balance and history of HTR and other tokens created at Hathor Network, so it's important for our users to easily see these information.

There are two options (i) adapt our desktop/mobile wallets to accept a 12-word-seed, so we can load the MetaMask private key in our own wallets, then the user would be able to see the balance and history of transactions, and (ii) create a web wallet that connects directly with MetaMask using snaps.

Even if the first option is the easiest one (and we can add this support anyway), I suggest we create a web wallet because users are used to interact with MetaMask using a web wallet and, most importantly, the seed would never leave the MetaMask environment.

### Features

The web wallet will eventually have all features of our desktop wallet but for the beginning I suggest we start with:

1. Load wallet using MetaMask
1. See balance
1. See history
1. See address

In the v2 we can add new features like:

1. Manage custom tokens (only to see balance and history)
1. Send transaction (with multiple outputs, data outputs, and custom tokens)

The network change is done inside MetaMask and we can get the correct network when loading the wallet.

We can use this opportunity to implement a new UI for the wallet that will be reused by our desktop wallet. This can easily be done creating a monorepo with a shared package with only components and the UI/UX that will be used by the web wallet and desktop wallet projects.

The web wallet is different from a dApp. In this case we must keep balance and history up to date handling real time update messages. Because of that we will load the wallet from MetaMask getting the xpub and storing a read only wallet object in the local storage. Any method that needs signature will connect with MetaMask again but all the other actions will be done locally using the xpub and websocket updates.

# Reference-level Explanation

## Snaps

### dApp call

The dApp will always interact with our snaps using a MetaMask RPC call, as in the snaps template code:

```
const data = await getSnapsProvider.request({
    method: 'wallet_invokeSnap',
    params: {
        snapId: <SNAP_ID>,
        request: {
            method: <RPC_METHOD>,
            params: <METHOD_PARAMS>
        }

    }
});


/**
 * Get a provider that supports snaps. This will loop through all the detected
 * providers and return the first one that supports snaps.
 *
 * @returns The provider, or `null` if no provider supports snaps.
 */
export async function getSnapsProvider() {
  if (typeof window === 'undefined') {
    return null;
  }

  if (await hasSnapsSupport()) {
    return window.ethereum;
  }

  if (window.ethereum?.detected) {
    for (const provider of window.ethereum.detected) {
      if (await hasSnapsSupport(provider)) {
        return provider;
      }
    }
  }

  if (window.ethereum?.providers) {
    for (const provider of window.ethereum.providers) {
      if (await hasSnapsSupport(provider)) {
        return provider;
      }
    }
  }

  const eip6963Provider = await getMetaMaskEIP6963Provider();

  if (eip6963Provider && (await hasSnapsSupport(eip6963Provider))) {
    return eip6963Provider;
  }

  return null;
}
```

In the template code it uses a utils method with context provider and hooks to implement it in a modular way, we can do it the same in a lib to share with dApp developers.

See [(i)](https://github.com/MetaMask/template-snap-monorepo/blob/main/packages/site/src/Root.tsx#L30), [(ii)](https://github.com/MetaMask/template-snap-monorepo/blob/main/packages/site/src/hooks/MetamaskContext.tsx), [(iii)](https://github.com/MetaMask/template-snap-monorepo/blob/main/packages/site/src/hooks/useRequest.ts), [(iv)](https://github.com/MetaMask/template-snap-monorepo/blob/main/packages/site/src/hooks/useInvokeSnap.ts), and [(v)](https://github.com/MetaMask/template-snap-monorepo/blob/main/packages/site/src/pages/index.tsx#L114).

### Snap code

#### RPC lib method call

After the dApp call above, the MetaMask snaps code will receive and identify each request with a code similar to the one below:

```
export const onRpcRequest: OnRpcRequestHandler = async ({
  origin,
  request,
}) => {
  switch (request.method) {
    case 'get_address':
      return handleGetAddress(request);
    case 'get_balance':
      return handleGetBalance();
    case 'send_transaction':
      return handleSendTransaction();
      ...
```

The handler method will be implemented using the RPC lib already developed for the Reown integration.

```
// This is a prompt handler expected by the RPC lib when the method requires user confirmation
const promptHandler = async (promptData, requestMetadata) => (
  await snap.request({
    method: 'snap_dialog',
    params: {
      type: 'confirmation',
      content: (
        <Box>
          <Text>
            The dApp {origin} wants to get your first empty address.
          </Text>
          <Text>
            Confirm the action below to continue.
          </Text>
        </Box>
      ),
    },
  })
);

// Get the Hathor node, corresponding to the path m/44'/280'/0'.
const hathorNode = await snap.request({
  method: "snap_getBip32Entropy",
  params: {
    curve: 'secp256k1',
    path: ['m', '44\'', '280\'', '0\''],
  },
})

const authHathorNode = await snap.request({
  method: "snap_getBip32Entropy",
  params: {
    curve: 'secp256k1',
    path: ['m', '280\'', '280\''],
  },
})

const accountPathXpriv = walletUtils.xprivFromData(Buffer.from(hathorNode.privateKey.substring(2), 'hex'), Buffer.from(hathorNode.chainCode.substring(2), 'hex'), hathorNode.parentFingerprint, hathorNode.depth, hathorNode.index, 'mainnet')
const authPathXpriv = walletUtils.xprivFromData(Buffer.from(hathorNode.privateKey.substring(2), 'hex'), Buffer.from(hathorNode.chainCode.substring(2), 'hex'), hathorNode.parentFingerprint, hathorNode.depth, hathorNode.index, 'mainnet')

let address = null;
const network = new Network('mainnet');
const walletServiceURL = 'https://wallet-service.hathor.network/';

wallet = new HathorWalletServiceWallet({
  requestPassword: () => Promise.resolve('123'),
  xpriv: accountPathXpriv,
  authxpriv: authPathXpriv,
  network,
  enableWs: false,
});

config.setWalletServiceBaseUrl(walletServiceURL);

await wallet.start({ pinCode: '123', password: '123' });

try {
  address = await rpcLib.getAddress(
    {
      method: 'htr_getAddress',
      params:
        {
          type: 'first_empty',
          network: 'mainnet'
        }
    },
    wallet,
    {},
    promptHandler
  );
} catch (e) {
  if (e instanceof PromptRejectedError) {
  } else {
    throw e;
  }
}

return address;
```

#### Home page

We can have a [home page](https://docs.metamask.io/snaps/features/custom-ui/home-pages/) to introduce and share more information about the snap with users.

#### On Install

We can [display a MetaMask dialog after the user installs our snap](https://docs.metamask.io/snaps/features/lifecycle-hooks/#2-run-an-action-on-installation) on it to share some more information, especially a documentation page.
 

# Task Breakdown

1. Wallet lib upgrade
  1. Phase 2: 5dd
1. Support for nano contracts in wallet service
  1. Waiting project ETA
1. Support all features in wallet lib class
  1. Nano contracts: 2dd
  1. Address methods: 2dd
  1. Get transaction methods: 2dd
1. Implement snaps method handling with UI
  1. Add utils/helpers/providers/hooks in RPC so dApps can easily call the MetaMask RPC with our method calls: 3dd
  1. All RPC methods with confirmation dialogs when needed: 8dd
  1. Home page: 0.5dd
  1. On install: 0.5dd
1. Web Wallet
  1. v1: 8dd
  1. v2: 8dd
1. Auditing
  1. External ETA

Total without Web Wallet: 23dd

Total: 39dd