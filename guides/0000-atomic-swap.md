### What is atomic swap

Atomic swap is a feature where two different users can exchange different tokens in the same transaction without any third party and any trust.

For instance, imagine you want to buy an NFT and pay with HTR. This can be done in a single transaction with no trust between the buyer and the seller.

### Operations flow

In the example described here we will assume that there is a central backend to coordinate the atomic swap. Let's assume we have two users (user1 and user2), each one with their own wallets (W1 and W2) in their devices and they want to swap token1 and token2, i.e. user1 will send token1 from W1 to W2 and user2 will send token2 from W2 to W2. See the image below.

![atomic swap wallets drawio](https://user-images.githubusercontent.com/3298774/145630746-ca4c0011-2e01-4574-8143-f3c64c214f4e.png)

The first thing to do is to define how many tokens will be exchanged, i.e. user1 will send X token1 to user2 and user2 will send Y token2 to user1. The atomic swap will happen in some steps that will be done in both wallets, where the users will need to exchange some information after each step. Below there is an image with each step and the explanation of them.

#### Step 1

User1 (that will send token1 and receive token2) will get utxos available of token1 and decide the ones that will use to send the amount needed to user2. Besides that, user1 will also decide the outputs that he wants to receive of token2 with amount and address. In this step it's important to also add any change output for token1, if needed.

#### Step 2

Step 2 is similar to the first one and can happen concurrently. User2 will get the utxos available of token2 and decide the ones that will be used. He will also define the outputs with amount and address to receive token1 from user1. In this step it's important to also add any change output for token2, if needed.

#### Step 3

The data on step 1 and step 2 will be used to create a transaction data with all inputs and outputs from both wallets. With this transaction data, we can start signing the inputs. The signature step will happen in two parts, one for user1 and one for user2.

#### Step 4

User1 will receive the data with all inputs and outputs with none of the inputs signed. He will check that the outputs of token2 that he expects to receive are correct (amount, address and timelock) and will sign the inputs that belong to his wallet. After that he will send his partial data to user2.

#### Step 5

User2 will receive the data with all inputs and outputs and only the inputs that belong to user1 will be signed. He will then check if the outputs of token1 that he expects to receive are correct (amount, address and timelock) and will sign the inputs that belong to his wallet. User1 don't need to worry about user2 changing the outputs before signing because, if that happens, his signature made on step 4 won't be valid anymore. After this step the transaction is ready for the final step.

#### Step 6

On step 6 the transaction will be completed with weight and timestamp, then will be sent to be mined and finally pushed to the network.

![atomic swap flow drawio (1)](https://user-images.githubusercontent.com/3298774/145630899-3d095687-061a-49f5-84f1-34e18a324660.png)

### Example code

This code below should work only with hathor wallet-lib v0.29.1 or above.

In the example code below we start two wallets, each one of them with a separated storage and we create a transaction that has inputs from both wallets and they exchange different tokens. In a real application, each wallet will be in a different device and this operation must be done asyncronously.

```
// You must call node in a folder that you have wallet-lib installed in node_modules
node

const hl = require('@hathor/wallet-lib')
const conn1 = new hl.Connection({network: 'mainnet', servers: ['https://node1.mainnet.hathor.network/v1a/']});
const conn2 = new hl.Connection({network: 'mainnet', servers: ['https://node1.mainnet.hathor.network/v1a/']});
const pin = '123';
const password = '123';
const w1 = new hl.HathorWallet({connection: conn1, password, pinCode: pin, seed: seed1});
const w2 = new hl.HathorWallet({connection: conn2, password, pinCode: pin, seed: seed2});

// Start wallets
w1.start()
w2.start()

// Verify is wallets are ready
w1.isReady()
w2.isReady()

// Get available utxos for each wallet and each token
w1.getUtxos({ token: token1 })
w2.getUtxos({ token: token2 })

// Create the tokens, inputs and outputs arrays with change outputs already
const tokens = [token1, token2]

// With the data from the get utxos method you will fill the inputs objects
// in the example below we will swap token1 from wallet1 with token2 from wallet2. token1 will be sent from a single utxo and token2 will be sent from 2 utxos.

const inputs = [{tx_id: txId1, index: index1, token: token1, address: address1}, {tx_id: txId2, index: index2, token: token2, address: address2}, {tx_id: txId3, index: index3, token: token2, address: address3}]

const outputs = [{address: address4, value: value1, token: token1, tokenData: 1}, {address: address5, value: value2, token: token2, tokenData: 2}]

const data = {tokens, inputs, outputs}
hl.transaction.completeTx(data);

// Sign inputs
const dataToSign = hl.transaction.dataToSign(data);
hl.storage.setStore(w1.store);
hl.transaction.signTx(data, dataToSign, pin);

hl.storage.setStore(w2.store);
hl.transaction.signTx(data, dataToSign, pin);

const transaction = hl.helpersUtils.createTxFromData(data, new hl.Network('mainnet'));
transaction.prepareToSend()

const sendTxObj = new hl.SendTransaction({ transaction })
await sendTxObj.runFromMining();
```
