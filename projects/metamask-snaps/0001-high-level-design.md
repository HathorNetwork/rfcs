# Motivation


# Guide-level Explanation

The metamask snaps works similar to the Wallet Connect, that we expose RPC methods to be used by dApps to sign transactions, get addresses, get balance, among other actions.

The metamask extension has its own private key and we can derive private keys in the Hathor Network derivation path. The main benefit of this is that we don't store any sensitive information from the user, we just use the metamask tools to derive our addresses.

## Running a wallet

Although it is possible to store data among RPC calls, real time update is not possible using snaps anymore (they removed support for [`endowment:long-running`](https://github.com/MetaMask/snaps/pull/1126) and [`websocket`](https://github.com/MetaMask/snaps/pull/1122/files)).

Because of that, the wallet service is the best choice for starting a wallet every time a RPC call is requested. Despite starting a websocket object, it's just for real time update and it's easy enough to implement a patch for removing this requirement.

### Using wallet service

The wallet service is still not ready to be used by the metamask snaps, especially because it currently still doesn't support nano contracts and its wallet lib in use is old.

#### Wallet lib upgrade

Wallet service is using wallet lib `v0.39.0`, and our latest wallet lib version is `v2.0.1`. We released 37 versions between them, with 2 major version changes. The approach for this upgrade will be to first upgrade to `v1.14.1`, which has the new storage scheme but without the big int support. Then, we will add this big int support, upgrading to `v2.0.1`. This last upgrade will need lot of changes in the code itself to support the big int values.

#### Support for nano contracts

This will be done by @andreabadesso in a different project, and its design is already in progress. It will contain the nano contract data to be pre calculated and stored in the wallet service database.

#### Support all features

We don't have all facade methods from the `HathorWallet` class implemented in the `HathorWalletServiceWallet` class. We must have full support, in order to be able to implement the RPC methods.

The least we need to have is:

1. `createAndSendNanoContractTransaction` and `createNanoContractTransaction` for nano contract support.
1. `isAddressMine` to use in nano contract validations.
1. `getAddressInfo` to use in the `getAddress` RPC method.
1. `getTx` and `getFullHistory` to use in some RPC methods.


NOTE: We currently don't support multisig in the wallet service facade but I think it's okay to continue with this for now.
 
## Implementation Phases

### RPC methods

The first version of the Hathor Snaps will have support for all RPC methods we currently support with Reown, and we will also support the `sendTx` method, which is not yet implemented because we are currently waiting for the generic template project to be finished.

The methods we will have support are:

1. `createToken`
1. `getAddress`
1. `getBalance`
1. `getUtxos`
1. `sendNanoContractTx`
1. `signOracleData`
1. `signWithAddress`
1. `sendTx`

### Web Wallet (v1)

It's common for projects implementing snaps support to have web wallets to interact with the metamask extension. We will also need to do that, otherwise users won't be able to load their wallets on their own, only through dApps.



### Web Wallet (final version)

## Auditing


# Task Breakdown

1. Wallet lib upgrade
  1. Phase 1: 5dd
  1. Phase 2: 5dd
1. Support for nano contracts in wallet service
  1. Waiting another project ETA
1. Support all features in wallet lib class
  1. Nano contracts: 1dd
  1. Address methods: 1dd
  1. Get transaction methods: 1dd
1. RPC methods
  1. All methods: 5dd
1. Web Wallet v1
  1. All features: 16dd
1. Web Wallet (final version)
  1. To be discussed
1. Auditing
  1. External ETA

Total: 34dd