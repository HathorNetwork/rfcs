# Summary
[summary]: #summary

Atomic swap is a feature where two different users can exchange different tokens in the same transaction without any third party and any trust. It consists of a single transaction that is either executed or not. Hence, the parties do not have to trust each other. If Alice receives Bob's token, then Bob will certainly receives Alice's token.

For instance, imagine you want to buy an NFT and pay with HTR. This can be done in a single transaction with no trust between the buyer and the seller.

# Motivation
[motivation]: #motivation

We already support atomic swap by design in the protocol, however with the growth of the NFT market we have already seen some use cases asking about that and one specific use case requested a code example with the steps on how to execute an atomic swap.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The basic idea of the atomic swap is that each user will fill the transaction with it's own inputs and outputs, then sign the transaction, so it can be mined and propagated to the network.

In the example described here we will assume that there is a central backend to coordinate the atomic swap, although there are other possibilities, e.g. the wallets might exchange messages through a broker (e.g. STUN server) and coordinate the swap.

Let's assume we have two users (Alice and Bob), each one with their own wallets (W1 and W2) in their devices and they want to swap token1 and token2, i.e. Alice will send token1 from W1 to W2 and Bob will send token2 from W2 to W1. See the image below.

![atomic swap wallets drawio](https://user-images.githubusercontent.com/3298774/145630746-ca4c0011-2e01-4574-8143-f3c64c214f4e.png)

The first thing to do is to define how many tokens will be exchanged, i.e. Alice will send X token1 to Bob and Bob will send Y token2 to Alice. The atomic swap will happen in some steps that will be executed by both wallets, where the users will need to exchange some information after each step.

The basic idea of the process is:

1. Wallet 1 and wallet 2 agree to exchange X token 1 and Y token2.
1. Wallet 1 sends the inputs and outputs (including any needed change output).
1. Wallet 2 sends the inputs and outputs (including any needed change output).
1. The transaction is assembled with the inputs and outputs and returned to both wallets.
1. Wallet 1 validates the transaction and sign its own inputs.
1. Wallet 2 validates the transaction and sign its own inputs.
1. The transaction is completed with weight and timestamp.
1. The transaction is mined and propagated to the network.

![atomic swap flow drawio](https://user-images.githubusercontent.com/3298774/145856624-19ebd557-f160-43d5-9267-60f55e6c4f74.png)

### Step 1

Alice (that will send token1 and receive token2) will get utxos available of token1 and decide the ones that will use to send the amount needed to Bob. Besides that, Alice will also decide the outputs that he wants to receive of token2 with amount and address. In this step it's important to also add any change output for token1, if needed. Wallet1 then sends this data to the backend service.

### Step 2

Step 2 is similar to the first one and can happen concurrently. Bob will get the utxos available of token2 and decide the ones that will be used. He will also define the outputs with amount and address to receive token1 from Alice. In this step it's important to also add any change output for token2, if needed. Then this data will also be sent to the backend.

### Step 3

The data on step 1 and step 2 will be used to create a transaction data with all inputs and outputs from both wallets. With this transaction data, we can start signing the inputs. The signature step will happen in two parts, one for Alice and one for Bob, and once again steps 4 and 5 can happen concurrently.

### Step 4

Alice will receive the data with all inputs and outputs with none of the inputs signed. He will check that the outputs of token2 that he expects to receive are correct (amount, address and timelock) and will sign the inputs that belong to his wallet. After that he will send the signed inputs back to the backend.

### Step 5

Bob will receive the data with all inputs and outputs with none of the inputs signed (if this step is being done concurrently with step 4). He will then check if the outputs of token1 that he expects to receive are correct (amount, address and timelock) and will sign the inputs that belong to his wallet. Alice don't need to worry about Bob changing the outputs before signing because, if that happens, his signature made on step 4 won't be valid anymore. After this step the transaction is ready for the final step.

### Step 6

On step 6 the backend already has the full data (all outputs and inputs with the signatures) to finish the operation. The transaction will be completed with weight and timestamp, then will be sent to be mined and finally pushed to the network.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This code below should work only with hathor wallet-lib v0.29.1 or above.

In the example code below we start two independent wallets, each one of them with a separated storage and we create a transaction that has inputs from both wallets and they exchange different tokens. In a real application, each wallet will be in a different device and this operation must be done asyncronously.

1. Call node in a folder that you have wallet-lib installed in `node_modules`
```bash
> node
```

2. Import the hathor wallet-lib and start both wallets that will take part on the atomic swap

```javascript
const hathorLib = require('@hathor/wallet-lib')
const conn1 = new hathorLib.Connection({network: 'mainnet', servers: ['https://node1.mainnet.hathor.network/v1a/']});
const conn2 = new hathorLib.Connection({network: 'mainnet', servers: ['https://node1.mainnet.hathor.network/v1a/']});
const pin = '123';
const password = '123';
const wallet1 = new hathorLib.HathorWallet({connection: conn1, password, pinCode: pin, seed: seed1});
const wallet2 = new hathorLib.HathorWallet({connection: conn2, password, pinCode: pin, seed: seed2});

// Start wallets
wallet1.start()
wallet2.start()
```

3. Verify that wallets are ready

```javascript
wallet1.isReady()
wallet2.isReady()
```

4. Get available utxos for each wallet and each token

```javascript
wallet1.getUtxos({ token: token1 })
wallet2.getUtxos({ token: token2 })
```

5. Create the tokens, inputs and outputs arrays with change outputs already
```javascript
const tokens = [token1, token2]

// With the data from the get utxos method you will fill the inputs objects
// in the example below we will swap token1 from wallet1 with token2 from wallet2. token1 will be sent from a single utxo and token2 will be sent from 2 utxos.
const inputs = [{
  tx_id: txId1,
  index: index1,
  token: token1,
  address: address1,
}, {
  tx_id: txId2,
  index: index2,
  token: token2,
  address: address2,
}, {
  tx_id: txId3,
  index: index3,
  token: token2,
  address: address3,
}];

const outputs = [{
  address: address4,
  value: value1,
  token: token1,
  tokenData: 1,
}, {
  address: address5,
  value: value2,
  token: token2,
  tokenData: 2
}];

const data = {
  tokens,
  inputs,
  outputs,
};

hathorLib.transaction.completeTx(data);
```

6. Sign the inputs with each wallet by loading their storage on the hathorLib

```javascript
const dataToSign = hathorLib.transaction.dataToSign(data);
hathorLib.storage.setStore(wallet1.store);
hathorLib.transaction.signTx(data, dataToSign, pin);

hathorLib.storage.setStore(wallet2.store);
hathorLib.transaction.signTx(data, dataToSign, pin);

const transaction = hathorLib.helpersUtils.createTxFromData(data, new hathorLib.Network('mainnet'));
transaction.prepareToSend()

const sendTxObj = new hathorLib.SendTransaction({ transaction })
await sendTxObj.runFromMining();
```
