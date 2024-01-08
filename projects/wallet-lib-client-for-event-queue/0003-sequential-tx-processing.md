# Summary
[summary]: #summary

Process each transaction as they arrive making atomic changes to balance and
other metadata in storage.

# Motivation
[motivation]: #motivation

The current history processing strategy is not efficient and is not scalable to
large numbers of transactions.
With the new event queue connection we have a solution for the issues that
required the current aproach so we can now create a more scalable solution.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When the wallet receives a new transaction or an existing transaction is
updated we need to update the calculated balances for the tokens on the
transaction since a token may have been spent or a transaction may have been
voided.
The current implementation simply erases all wallet metadata (balances, list of
utxos, etc) and calculates it by iterating over all transactions. While this
approach was needed in the past, due to how the event system was setup it would
take longer times with larger wallets.

We will also have to consider the load of processing the history of transactions
on startup and when we are re-joining the network if the connection goes offline.

## Processing a single transaction

The core of the implementation will be a method to process a single transaction.
This method will first check if a transaction with the same id is already on
storage, this means we are updating an existing transaction.

### New transactions

If this is a new transaction but it is voided, we can ignore it and return null.
This should not happen but we will treat it as a no-op.

Otherwise we need to iterate over the outputs of the transaction and for each one we need
to check if the address is from our wallet, if it is we save the:
- tokens involved in the transaction
- token balance
- addresses involved in the transaction (only the ones from our wallet)
- address balance (by token)
- utxos, for unspent outputs
- highest address index

We only need to check the outputs since any change of the inputs should be
calculated by the update of their transaction.

With the information above we can update the necessary data on the storage.

### Update existing transactions

We should first check if the transaction is being voided or unvoided.
Voiding means the `voided_by` field is `null` on storage and not `null` on the transaction.
Unvoiding means the `voided_by` field is `null` on the transaction and not `null` on storage.

When voiding a transaction we need to undo any impact it has on the balance of
the wallet and the other metadata.
This means calculating the same data as a new transaction but we should make the
opposite changes (balances will be removed, etc).

When unvoiding a transaction we can treat it as a new transaction since its
effect on the wallet was already removed when it was voided.

If the `voided_by` field is `null` we can iterate on the outputs and calculate
the changes the update will have on the wallet.
This is different from the new transaction because we will need to compare the
output from the transaction to the output from the storage.
Since a transaction cannot change any output data we can just check the metadata
`spent_by` which will indicate if the output is being spent or unspent.
With this information we can calculate the balance change, since the other
metadata will not change.

### Output for a single transaction

```ts
interface ITxMetadataEffect {
  was_voided: boolean;
  addresses: string[];
  tokens: string[];
  // A map of address to balance (indexed by token uid)
  address_balance: Record<string, Record<string, IBalance>>;
  // A map of token uid to balance
  token_balance: Record<string, IBalance>;
  add_utxos: IUtxo[];
  del_utxos: IUtxo[];
  highest_address_index: number;
}
```

## Processing a stream of transactions

To process a list or stream of transactions we will process each transaction,
make atomic changes on the storage then continue down the stream.

We will calculate the `ITxMetadataEffect` for the transaction, then apply the
changes to the storage.
The process will be different depending on `was_voided`.

### `was_voided === false`

This means the transaction was not voided, so we will act on the `ITxMetadataEffect`.
This is the process for new transactions and updates.

- `addresses`: We should add 1 to the `numTransactions` for each address (or set 1)
- `tokens`: We should add 1 to the `numTransactions` for each token (or set 1)
- `add_utxos`: Add the utxos to the storage
- `del_utxos`: Remove the utxos to the storage
- `highest_address_index`: Check if we have a new highest address index and set it.
- `token_balance`: for each token, add the balance to the existing balance.
- `address_balance`: for each address, add the balance to the existing balance.

If `highest_address_index` is higher than the wallet's `lastUsedAddressIndex` we
will set it as the `lastUsedAddressIndex`. If it is higher than
`current_address_index` we will set the current address index to
`min(lastLoadedAddressIndex, highest_address_index + 1)`.

If we had to set the `numTransactions` to 1 for any token, this means the token
just entered the wallet, this also means that we may have to fetch the token data
from the fullnode.

### `was_voided === true`

This means the transaction was voided, so we will remove the `ITxMetadataEffect`.

- `addresses`: We should subtract 1 from `numTransactions` for each address
  - If `numTransactions` is 1, this means that the address just became unused.
- `tokens`: We should subtract 1 from `numTransactions` for each token
  - If `numTransactions` is 1, this means that the token was removed from the wallet.
- `add_utxos`: Add the utxos to the storage.
- `del_utxos`: Remove the utxos to the storage.
- `token_balance`: for each token, remove the balance from the existing balance.
- `address_balance`: for each address, remove the balance from the existing balance.
