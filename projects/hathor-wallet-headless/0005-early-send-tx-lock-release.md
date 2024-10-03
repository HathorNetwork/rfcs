- Feature Name: early_send_tx_lock_release
- Start Date: 2024-09-17
- Author: Andre Carneiro <andreluizmrcarneiro@gmail.com>

# Summary
[summary]: #summary

Release the send-tx lock after the transaction is prepared and before it is mined.

# Motivation
[motivation]: #motivation

The only impediment to running 2 send transaction simultaneously is trying to choose the UTXOs and selecting the same ones.
Meaning it is safe to release the send-tx lock after the transaction is chosen.
This would allow sending more transactions from the same headless wallet instance.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Wallet-lib

We should aim to affect all types of wallets capable of sending transactions (P2SH, P2PKH, HSM, Fireblocks, etc.).
The wallet facade has some methods that are meant to prepare and send transactions, these methods return a promise that resolves when the transaction is accepted by the network.
This promise is the return of either `run` or `runFromMining` from the `SendTransaction` facade.

The methods that send transactions are:

- `sendManyOutputsTransaction`
- `handleSendPreparedTransaction`
- `consolidateUtxos`
- `sendTransaction`
- `createNewToken`
- `mintTokens`
- `meltTokens`
- `delegateAuthority`
- `destroyAuthority`
- `createNFT`
- `createNanoContractTransaction`
- `createAndSendNanoContractTransaction`

We will create new methods that return the `SendTransaction` instance instead and these methods will be changed to use the new methods to avoid code duplication.
This will give the caller more control over the execution of the "send transaction" process.

## Headless send-tx lock

We will change the call of any methods listed above to use the new method.
This will enable the headless to unlock the send-tx lock before mining the transaction.

The send-tx process can be seen as divided in 3 steps:

- `prepare-tx`
  - Choose UTXOs and create a `Transaction` instance
  - `SendTransaction` can be instantiated with the `Transaction` instance
- `mine-tx`
  - Lock utxos from the `Transaction` instance
  - Send transaction to a tx-mining service and get the `nonce`
- `push-tx`
  - Send the transaction to the fullnode.

The headless will have to manually lock the UTXOs after the `prepare-tx` step so the UTXOs are not chosen by another call to create a transaction.

Some of the wallet-lib methods initiate a `SendTransaction` instance from the expected outputs and others from a prepared `Transaction` instance.
The caller can still identify if the instance has a prepared transaction or not, meaning the execution steps are:

- If `sendTransaction.transaction` is `null`
  - run `prepareTx`
- Lock UTXOs with `updateOutputSelected`
- Release the send-tx lock.
- call `.runFromMining()` and return the result.

Since these steps do not depend on the method called we can move this to an util method to be used on all routes.

## HSM special case

The HSM does not allow simultaneous connections so we should use the global send-tx for any HSM wallet.
Since we release the connection after the transaction is signed the early lock release will be safe if all HSM wallets use the same lock.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Peek internals with an EventEmitter

The headless could inject an `EventEmitter` and receive updates on what is happening during the "send transaction" process.
These updates could be used to release the send-tx lock and would not change the facade methods types.
Returning the `SendTransaction` instance was chosen because it gives the caller more control over the process and leaves less oportunities for memory leaks.

## Create a Task facade to control the process

The "Task facade" would be initiated with the steps to execute and the caller would be able to run code between steps or just run all steps.
This allows oportunities to abort or release the lock after the `prepare-tx` step and can be useful for other processes as well.

This was discarded due to the overcomplication and time to develop.

# Unresolved questions

## Finite UTXO selection

Allowing transactions to choose UTXOs faster can eventually make all available UTXOs chosen before the transactions are sent.
This fails the next attempt to send a transaction with a "Not enough tokens" error even if the wallet may have the balance.

If a wallet is meant to send many transactions in a burst and it wants to avoid this it should spread the tokens on as much UTXOs as possible.
It will not prevent this error, but minimize since less tokens should be on change outputs that have yet to be sent.

# Task Breakdown
[task-breakdown]: #task-breakdown

- [ ] Create new lib methods (2 dev-days).
- [ ] Change headless to use the new lib methods (1 dev-day).
- [ ] Change headless to release the send-tx lock early (1 dev-day).
- [ ] Make all HSM wallets use the same lock (1 dev-day).

Total: 5 dev-days.
