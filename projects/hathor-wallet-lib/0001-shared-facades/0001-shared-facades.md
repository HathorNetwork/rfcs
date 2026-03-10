- Feature Name: wallet-lib-shared-facades
- Start Date: 2026-01-21
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: tulio@hathor.network

# Summary
[summary]: #summary

The Wallet Lib facades for the Fullnode and the Wallet Service should share a subset of the most important methods, with the same parameters and output data format.

This will cause breaking changes that will require a major version update on the lib.

# Motivation
[motivation]: #motivation

At the moment, both facades are similar, but not equal. This causes duplicated test suites, requires additional time and attention from the developers and ultimately introduces bugs on the Wallet Lib clients when they need to switch between facades.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This document serves as a mapping, reference and roadmap to mitigate this discrepancy between the facades. It is expected that many PRs will be opened to solve it entirely, and they will need a centralized end-goal to be consistent.

To achieve this desired unified state it's necessary to change the contracts for some methods: both in the wallets and in their shared interface `IHathorWallet`, which is now outdated. An [initial investigation](./shared-contract-investigation.md) was made to understand the necessary changes, and they will be summarized below.

- Change methods from `sync` to `async` on the Wallet Service facade
- Modify `IHathorWallet` return types that currently do not match the fullnode facade
- Create types for the parameters of both facades, facilitating visual inspection from devs
- Decide on `UTXO` and `Tx History` objects that currently differ, and are used by most of the applications
- Decide which subset of methods will consist of the `IHathorWallet`, and leave each facade to have its own complementary methods
- Add methods that are present in both facades but not in the `IHathorWallet`, such as `getAuthorityUtxo()`
- Create a `shared_types.ts` on the root folder, instead of using types from each facade folder

In the desired end-state it should be possible for a hypothetical client implementation to call methods from a Fullnode or Wallet Service facades interchangeably during its operation, with no adaptation to their inner workings.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 1. Async/Sync Signature Alignment

The following methods must become async in both facades. The Wallet Service facade currently returns synchronous values, while the Fullnode facade returns Promises. This is a critical part that must be addressed first.

### 1.1 Address Methods

```typescript
// Before (WalletServiceWallet - sync)
getCurrentAddress(options?: { markAsUsed: boolean }): AddressInfoObject;
getNextAddress(): AddressInfoObject;

// After (both facades - async)
getCurrentAddress(options?: { markAsUsed: boolean }): Promise<AddressInfoObject>;
getNextAddress(): Promise<AddressInfoObject>;
```

### 1.2 Lifecycle Methods

```typescript
// Before (IHathorWallet - sync)
stop(): void;

// After (IHathorWallet - async, matching both implementations)
stop(): Promise<void>;
```

## 2. Unified Return Types
The second most important change is to unify the types that are returned for the main wallet operation methods. Without this, adaptations need to be implemented at every interaction point, which lead to unnecessary complexity and opens room for bugs.

### 2.1 Address Info Object

Both facades will return a consistent `AddressInfoObject`:

```typescript
interface AddressInfoObject {
  address: string;
  index: number;
  addressPath: string;
  info?: string; // Optional, WalletService-specific metadata
}
```

**Corner case:** `index` is `number | null` in Fullnode facade. This will be changed to always `number`, as unknown/unindexed addresses are not possible in the scope of this method.

### 2.2 Authority UTXO Type

Create a unified `AuthorityUtxo` type for `getMintAuthority()`, `getMeltAuthority()`, and `getAuthorityUtxo()`:

```typescript
interface AuthorityUtxo {
  txId: string;
  index: number;
  address: string;
  authorities: OutputValueType;
  // Optional fields present when available from storage
  tokenId?: string;
  value?: OutputValueType;
  timelock?: number | null;
}
```

Both facades will return `Promise<AuthorityUtxo[]>`. The Fullnode facade currently returns `IUtxo[]` (richer), while WalletService returns `AuthorityTxOutput[]` (minimal). The unified type uses the intersection of required fields, with optional enrichment.

### 2.3 Transaction Return Types

```typescript
// Standardize to non-nullable return
sendTransaction(...): Promise<Transaction | null>;
sendManyOutputsTransaction(...): Promise<Transaction | null>;
```

**Corner case:** Fullnode facade returns `null` when `startMiningTx: false`. This null possibility will be enforced in the other facade as well.

## 3. Interface Method Additions

From now on we have improvements that enhance readability and maintainability, but no longer are bug-inducing code structures. The third step would be to add methods that exist in both facades but are missing from `IHathorWallet`:

```typescript
interface IHathorWallet {

  // Authority methods (currently missing)
  getMintAuthority(tokenUid: string, options?: AuthorityOptions): Promise<AuthorityUtxo[]>;
  getMeltAuthority(tokenUid: string, options?: AuthorityOptions): Promise<AuthorityUtxo[]>;
  getAuthorityUtxo(tokenUid: string, authority: 'mint' | 'melt', options?: AuthorityOptions): Promise<AuthorityUtxo[]>;

  // State methods (currently missing)
  isReady(): boolean;

  // Cleanup methods (currently missing)
  clearSensitiveData(): void;
  isHardwareWallet(): boolean;
}

// Will be declared on the dedicated shared types file
interface AuthorityOptions {
  many?: boolean;
  skipSpent?: boolean; // Replaces only_available_utxos
  filterAddress?: string; // Replaces filter_address (camelCase)
}
```

## 4. Typed Options Parameters

Replace untyped `options` with explicit types:

```typescript
interface CreateTokenOptions {
  address?: string;
  changeAddress?: string;
  createMint?: boolean;
  mintAuthorityAddress?: string;
  allowExternalMintAuthorityAddress?: boolean;
  createMelt?: boolean;
  meltAuthorityAddress?: string;
  allowExternalMeltAuthorityAddress?: boolean;
  data?: string;
  pinCode?: string;
}

interface SendTransactionOptions {
  token?: string;
  changeAddress?: string;
  pinCode?: string;
}

interface SendManyOutputsOptions {
  inputs?: Array<{ txId: string; index: number }>;
  changeAddress?: string;
  pinCode?: string;
}
```

## 5. Shared Types File Structure

Create `src/shared_types.ts` containing:

```
src/
├── shared_types.ts          # New: All shared interface types
├── new/
│   └── wallet.ts            # HathorWallet (Fullnode facade)
│   └── types.ts             # Fullnode-specific types only
└── wallet/
    └── wallet.ts            # HathorWalletServiceWallet
    └── types.ts             # WalletService-specific types only
```

The `IHathorWallet` interface moves from `src/wallet/types.ts` to `src/shared_types.ts`, along with all types used in interface method signatures.

## 6. Methods Remaining Facade-Specific

The following methods will NOT be added to `IHathorWallet` and remain facade-specific:

**Fullnode-only:**
- `buildTxTemplate()`, `runTxTemplate()` - Transaction template system
- `getFullHistory()` - Large dataset, different storage models
- `consolidateUtxos()` - Requires local UTXO management
- `setGapLimit()`, `getGapLimit()` - Local address derivation control
- `syncHistory()`, `reloadStorage()` - Local storage operations
- Multisig methods - Different signing workflows

**WalletService-only:**
- Service-specific connection management
- Polling configuration

## 7. Implementation Order

This is a suggested approach, but many of those tasks can be executed in parallel. The shared tests, for example, can be started immediately by using temporary adapters such as on [Wallet Lib 1032](https://github.com/HathorNetwork/hathor-wallet-lib/pull/1032).

1. Create `shared_types.ts` with unified types
2. Update `IHathorWallet` interface with new signatures
3. Modify WalletServiceWallet sync methods to async
4. Update return types in both facades to match interface
5. Add missing methods to interface
6. Update existing tests to use shared test patterns
7. Create shared test suite for interface compliance

## Null and Undefined types
A note that influences decisions that impact all items described above: the main questions to resolve through this RFC are the actual method signatures, deciding which properties are mandatory, which will be optional.

A special attention should be paid to the `null` parameter, which is used in many places as a valid replace for `undefined`.

This is an antipattern present in many places inside the Wallet Lib that use `null` and `undefined` interchangeably. They are meant to have different semantics: for example, an optional parameter should be defined as `parameter?: type` and not `parameter?: type | null`. This adds noise to the typing that cascade up and down the caller stacks.

### Creation of a `beta` version
When the breaking change implementations start, we should create a `beta-release` branch to the Wallet Lib repository, where we could release multiple PRs, publish them to `npm` and only use those in our own clients ( Desktop/Mobile/Headless/Web Wallets ).

This is a way to allow for our entire flow of reviews and internal tests to complete before exposing the new changes to the external public. It will also avoid accumulating a large code diff to `master` that would require a huge review effort and further postpone the refactoring effort.

# Drawbacks
[drawbacks]: #drawbacks

The only reasons not to do this would be to preserve backwards compatibility with existing clients, or to dedicate development priority elsewhere.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The facades could be kept separate and a wrapper `WalletWrapper` could be developed to abstract away their differences. This would allow for shared integration tests and a simpler developer experience, while the internal work is postponed for a better time.

The downside of this alternative is having additional work and complexity in order to avoid a synchronization work that will have to be done eventually.

By not implementing this shared facade, we keep the responsibility on the client to know what facade is being used and the details of each one separately in the client code.

# Prior art
[prior-art]: #prior-art

The objective of the `IHathorWallet` interface was to create a shared contract for both facades, but without automated testing it was natural to drift away from a compliance strict enough for a full TypeScript validation.

The main proposal of this RFC is to bring this interface up-to-date with the current state of the wallets, improving it albeit with breaking changes.


# Future possibilities
[future-possibilities]: #future-possibilities

An effort could be made to break the facade files into multiple pieces with common group responsibilities that could be shared between the two facades. This would make it easier for LLMs to interact with those files, instead of having two facades with over 3k lines that do not fit their context windows.

We could also re-think of the facades themselves to make managing the connections better, taking advantage of the breaking changes being implemented on the `beta` branch to improve on more parts of the code that would cause those breaks.

# Annex

## Additional changes
Taking advantage of the amount of breaking changes that will be implemented in the `v3`, a few other unrelated changes are also going to be pushed with this design.

### SendTransaction.run() parameters
[Link to the code](https://github.com/HathorNetwork/hathor-wallet-lib/blob/56b81dacff41f1546e1a93a359f7ce1313ace7ef/src/wallet/types.ts#L490-L493) Currently both parameters are optional, but if you want to inform the second, the first must be informed as empty. A change will turn them from named parameters to a single object containing all optional ( see [this comment from PR 1022](https://github.com/HathorNetwork/hathor-wallet-lib/pull/1022#discussion_r2878658323))
