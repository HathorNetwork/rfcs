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

## Wallet-lib autorun

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

These methods (aside from `handleSendPreparedTransaction`) receive an `options` argument, we will add an `autorun` boolean that defaults to `true`.
When `autorun` is `false` the method should return the `SendTransaction` instance instead.
This will give the caller more control over the execution of the "send transaction" process.

## Headless send-tx lock

We will change the call of any methods listed above to not use autorun.
This will enable the headless to unlock the send-tx lock before mining the transaction.
Any UTXOs locked during the `prepare-tx` step will not be chosen if another method tries to send a transaction since the `SendTransaction` facade marks them as 'selected as input'.

Some of the wallet-lib methods initiate a `SendTransaction` instance from the expected outputs and others from a prepared `Transaction` instance.
The caller can still identify if the instance has a prepared transaction or not, meaning the execution steps are:

- If `sendTransaction.transaction` is `null`
  - run `prepareTx`
  - Release send-tx lock.
- If `sendTransaction.transaction` is not `null`
  - Release the send-tx lock.
- call `.runFromMining()` and return the result.

Since these steps do not depend on the method called we can move this to an util method.

## HSM special case

The HSM does not allow simultaneous connections so we should use the global send-tx for any HSM wallet.
Since we release the connection after the transaction is signed the early lock release would be safe if all HSM wallets use the same lock.

# Drawbacks
[drawbacks]: #drawbacks

This adds another option on the facade methods, making the facade more convoluted.

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

# Task Breakdown
[task-breakdown]: #task-breakdown

- [ ] Add the autorun option on wallet-lib (2 dev-days).
- [ ] Change headless to use the `autorun=false` (0.5 dev-day).
- [ ] Change headless to release the send-tx lock early (0.5 dev-day).
- [ ] Make all HSM wallets use the same lock (1 dev-day).

Total: 4 dev-days.
