- Feature Name: nano-contracts-support
- Start Date: 2025-01-02
- Author: @andreabadesso

# Summary

This document describes the integration of nano-contracts support into the wallet-service, enabling tracking and indexing of nano-contract transactions, including caller information and providing proxy endpoints for fullnode interaction.

# Motivation

The Hathor Network is introducing nano-contracts. To maintain complete transaction history for wallets and addresses, we need to adapt our indexing service to handle these new transactions and provide appropriate APIs for interacting with nano-contracts.

# Guide-level explanation

Currently, the wallet-service syncs with the fullnode through reliable integrations, which sends events in the exact order they occur. The daemon processes these events through a state machine (SyncMachine) that handles different types of events like new transactions, metadata changes, and voided transactions.

For nano-contracts, the main change is tracking the caller's relationship with the transaction, especially when they are not directly involved in token transfers. When a nano-contract transaction is received (identified by its version), we'll:

1. Process normal inputs/outputs as usual
2. Check if the caller is involved in inputs/outputs
3. If not, explicitly add the transaction to their history
4. Forward any contract-specific API calls to the fullnode

This means that if address is the caller in a contract call that moves tokens from X to Y (e.g. a deposit action), all three addresses will see this transaction in their history:
- X and Y through normal UTXO tracking
- Z through explicit caller tracking

The existing transaction history endpoints will work with nano-contracts without changes:
- `GET /wallet/history` - Returns all transactions involving the wallet, including:
  * Transactions where the wallet's addresses are inputs/outputs
  * Transactions where the wallet's addresses are callers
- `GET /wallet/transactions/{txId}` - Returns details of a specific transaction
  * Works for both regular and nano-contract transactions
  * Includes all token movements and metadata

From the user's perspective, they will be able to:
- See all their contract interactions in their history, whether they moved tokens or just called the contract
- Create and execute contracts through the wallet-service API
- Query contract state through proxied fullnode APIs

# Reference-level explanation

## Database Changes

No schema changes are required. The existing tables (`address_tx_history` and `wallet_tx_history`) will be used to track contract interactions, but we need to explicitly add entries for callers when they are not involved in inputs/outputs.

Important implementation notes:
1. The `updateAddressTablesWithTx` function needs to be modified to handle zero balances efficiently:
   - Keep updating the `address` table for transaction count
   - Skip `address_balance` updates when all values (balance and authorities) are zero
   - Always add entries to `address_tx_history`, even for zero balances

2. The `voided` column in history tables applies to callers as well - if a nano-contract transaction is voided, all history entries (including the caller's) must be marked as voided

## Daemon Changes

### SyncMachine Updates
The `handleVertexAccepted` service will be enhanced to handle nano-contract transactions:

1. Detect nano-contract transactions (by version)
2. Extract the caller information from the transaction metadata
3. If the caller is not involved in any inputs or outputs:
   - Use `updateAddressTablesWithTx` with a zero balance to add the transaction to their history
   - The function will handle incrementing transaction count and adding history entries

Example flow:
```typescript
// In handleVertexAccepted
if (isNanoContractTx(metadata)) {
  const { caller } = metadata;
  const addresses = new Set([
    ...inputs.map(i => i.address),
    ...outputs.map(o => o.address)
  ]);

  // If caller is not in inputs or outputs, add to history
  if (!addresses.has(caller)) {
    // Create a map with zero balance for the HTR token
    const zeroBalanceMap = {
      [caller]: TokenBalanceMap.fromStringMap({
        '00': { unlocked: 0, locked: 0 }
      })
    };

    await updateAddressTablesWithTx(mysql, txId, timestamp, zeroBalanceMap);
  }
}
```

Example scenarios:
```
Scenario 1: X (caller) sends to Y
- X appears in history (input)
- Y appears in history (output)

Scenario 2: Z (caller) triggers X to send to Y
- Z appears in history (zero balance entry via updateAddressTablesWithTx)
- X appears in history (input)
- Y appears in history (output)

Scenario 3: Y (caller) triggers X to send to Y
- X appears in history (input)
- Y appears in history (output)
```

## Wallet Service Changes

### Proxy Endpoints

The wallet-service will transparently proxy all nano-contract related requests to the fullnode instead of reimplementing the state tracking functionality. This decision was made because:
1. The fullnode already maintains and optimizes contract state tracking
2. The fullnode provides APIs to query state at specific blocks
3. Reimplementing state tracking would require significant complexity and storage overhead

Implementation:
```
/api/nano_contract/* -> fullnode's /v1a/nano_contract/*
```

All nano contract endpoints will be proxied transparently, maintaining the same interface as the fullnode. The wallet-service will:
- Forward all requests to the corresponding fullnode endpoint
- Preserve request parameters and body
- Handle errors appropriately (including 404s)
- Maintain proper authentication and rate limiting

# Drawbacks

1. Additional storage requirements for tracking caller information
2. Increased complexity in transaction processing
3. Need for careful handling of contract-related errors

# Rationale and alternatives

The proposed design:
- Minimizes changes to existing tables
- Reuses existing sync mechanisms
- Maintains backward compatibility
- Provides clear separation between contract and regular transactions

Alternative approaches considered:

1. Storing nano-contract state locally in wallet-service
   - This would require new tables to store:
     * Contract state at each block height
     * Contract execution history
     * Blueprint information and metadata
   - Drawbacks:
     * Significant storage overhead (storing state for every contract)
     * Complex state management (handling reorgs, voided transactions)
     * Need to replicate fullnode's state validation logic
     * Risk of state divergence between fullnode and wallet-service
     * Additional indexing overhead for state queries
   - Benefits:
     * Potentially faster queries for frequently accessed contracts
     * Ability to build custom indexes for specific contract types
   - Rejected because:
     * The storage and complexity costs outweigh the benefits
     * The fullnode already optimizes state storage and access
     * State queries are not as frequent as balance/history queries

2. Separate tables for nano-contract transactions
   - Would require duplicating transaction data
   - Increases complexity of history queries
   - Makes it harder to maintain consistency
   - Rejected due to unnecessary data duplication
