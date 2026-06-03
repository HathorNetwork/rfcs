- Feature Name: global-balances
- Start Date: 2026-03-10
- RFC PR:
- Hathor Issue:
- Author: Jan Segre <jan@hathor.network>

# Summary
[summary]: #summary

This RFC defines a global address-balance map that co-exists with UTXOs. The new balance domain is keyed by `(address, token)` and is intended to be the standard shared balance space for user-owned funds that contracts use.

This RFC also defines `TransferHeader`, a transaction header that bridges UTXO balances and global address balances. A transaction may use UTXOs, global-balance debits and credits, or both. Pure global-balance transfers are first-class transactions.

Nano contracts are one consumer of this shared balance domain. Contracts keep their own treasury as contract-local state, but they may read global address balances and transfer contract-owned funds to the global address-balance map.

# Motivation
[motivation]: #motivation

Today, Hathor has two unsatisfactory extremes for programmable balances.

- UTXOs are the native settlement format, but they force transaction authors to know the concrete input/output shape before execution.
- Contract-local balance maps are flexible, but each contract reinvents the same user-ledger logic and exposes a different interface to wallets, explorers, and other contracts.

The missing piece is a shared, protocol-level balance domain for user-owned programmable funds.

This RFC introduces that domain without replacing the UTXO model:

- UTXOs remain the native settlement object.
- Global address balances become the shared programmable balance space.
- `TransferHeader` becomes the explicit bridge between the two.

The goal is to let Hathor support account-like programmable balances where they help, while still keeping UTXOs as the canonical settlement layer.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Mental model

After this RFC, Hathor has two protocol balance domains and one contract-local domain:

- UTXO balances: value locked in regular transaction outputs.
- Global address balances: protocol-owned balances keyed by `(address, token)`.
- Contract treasury balances: value owned by a nano contract in its own local state.

Global address balances are for user-owned funds. Contract treasury is for contract-owned funds. They are not the same thing and must not be merged conceptually.

`TransferHeader` is the bridge between UTXOs and the global address-balance map:

- transfer inputs debit the global address-balance map
- transfer outputs credit the global address-balance map
- regular inputs and outputs keep working as they do today

So a transaction may:

- move value from UTXOs into the global map
- move value from the global map into UTXOs
- move value entirely inside the global map
- combine those movements with nano-contract execution

This is not a pure account model. UTXOs remain the settlement format. The global map is an additional protocol balance space for cases where explicit UTXO planning is too rigid.

## A note on decimal precision

This feature is built on top of [Token Amount V2](../projects/token-amount-v2/0002-token-amount-v2.md), the upgrade from 2 to 18 decimal places. Every amount in the global map is an 18-decimal (V2) base-unit integer: the examples below write human-readable values such as `4 HTR`, but on the wire and in storage that is `4 * 10^18` base units. The global balance map is 18-decimal native; it does not exist for the legacy 2-decimal (V1) representation. See [Decimal precision](#decimal-precision-token-amount-v2) for the full rules.

## Examples

### UTXO to global balance

Alice wants to move 4 HTR from UTXO space into her global balance.

```text
tx inputs:
- spend 10 HTR UTXO owned by Alice

tx outputs:
- 6 HTR change back to Alice

transfer_header outputs:
- credit 4 HTR to Alice
```

After the transaction, Alice has 4 HTR in `GlobalAddressBalance[(Alice, HTR)]`.

### global balance to UTXO

Alice wants to withdraw 3 HTR from her global balance into a regular UTXO.

```text
transfer_header addresses:
- [0] Alice, seqnum = 7, script = Alice signature

transfer_header inputs:
- debit 3 HTR from addresses[0]

tx outputs:
- 3 HTR regular output to Alice
```

No UTXO input is required because the funding source is the global address-balance map.

### global balance to global balance

Alice wants to transfer 5 HTR to Bob without materializing any UTXO.

```text
transfer_header addresses:
- [0] Alice, seqnum = 8, script = Alice signature

transfer_header inputs:
- debit 5 HTR from addresses[0]

transfer_header outputs:
- credit 5 HTR to Bob

tx inputs:
- none

tx outputs:
- none
```

This is still a valid transaction. It simply settles entirely in the global address-balance map.

### caller funds a contract call from global balance

Alice wants to call a nano contract and pay 5 HTR from her global balance into the contract treasury.

```text
nano_header:
- caller = Alice
- action = deposit 5 HTR into contract treasury
- method = deposit_for_order()

transfer_header addresses:
- [0] Alice, seqnum = 9, script = Alice signature

transfer_header inputs:
- debit 5 HTR from addresses[0]

tx inputs:
- none

tx outputs:
- none
```

In this RFC, that is the only way a contract may consume user funds from the global map:

- the debited address is the caller
- the debit is predeclared in the transaction
- the contract cannot invent new user debits during execution

### contract settles into the global balance map

A contract already owns 20 HTR in its treasury and wants to pay 3 HTR to Bob's global balance.

```python
self.syscall.transfer_to_address(bob, 3, HTR)
```

After execution:

- contract treasury decreases by 3 HTR
- `GlobalAddressBalance[(Bob, HTR)]` increases by 3 HTR

No user global balance is debited by the contract in this flow. The source of funds is the contract treasury itself.

## How contract authors should think about this

Contract authors should treat global address balances as the shared protocol balance space for user-owned programmable funds.

If a contract needs its own funds, escrows, or internal treasury, those remain contract-local. If a contract wants to read a user's shared programmable balance or settle contract-owned value to a user, it should use the global map instead of reinventing a contract-specific address ledger.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## State model

The protocol maintains two global-state tables for this feature:

- `GlobalAddressBalance[(address, token)] -> amount`
- `GlobalAddressSeqnum[address] -> last_seqnum`

One concrete encoding is:

- balance key: `ADDRESS_BALANCE || address || token_id`
- seqnum key: `ADDRESS_SEQNUM || address`

Missing balance entries imply zero. Missing seqnum entries imply `-1`, so the first valid seqnum for an address is `0`.

Only regular user addresses participate in the global address-balance map. Contract IDs are not valid entries in either table.

That implies:

- `TxTransferOutput.address` must be a regular address
- `InputAddress.address` must be a regular address
- `transfer_to_address(address, amount, token)` must reject contract IDs

Contracts still own funds through contract-local treasury state, not through entries in `GlobalAddressBalance`.

## Decimal precision (Token Amount V2)
[decimal-precision-token-amount-v2]: #decimal-precision-token-amount-v2

This feature is specified directly on top of [Token Amount V2](../projects/token-amount-v2/0002-token-amount-v2.md), which raises token precision from 2 to 18 decimal places. It is 18-decimal native from the start; there is no 2-decimal (V1) form of the global balance map.

The rules are:

- Every amount in this RFC — `GlobalAddressBalance[(address, token)]` values, `TxTransferInput.amount`, `TxTransferOutput.amount`, and the syscall amounts — is a **V2 (18-decimal) normalized base-unit integer**. This matches the full node's internal accounting, which is always V2-normalized; V1 values elsewhere are scaled up by the normalization factor `10^16` (Token Amount V2, I7/I9).
- Global balance and seqnum entries live in the same patricia-trie used for nano contract state, which Token Amount V2 re-serializes to V2 (I18). Amounts are stored using the trie's existing internal integer encoding (LEB128); this RFC introduces no new encoding.
- This feature **requires Token Amount V2 to be active**. It is gated behind its own Feature Activation process whose activation must not precede V2. Before V2 is active there are no V2 transactions, so no `TransferHeader` can be accepted.

## Transaction format

Transactions may include at most one `TransferHeader`.

```text
TransferHeader {
    addresses: list[InputAddress]
    inputs: list[TxTransferInput]
    outputs: list[TxTransferOutput]
}

InputAddress {
    address: Address
    seqnum: int
    script: bytes
}

TxTransferInput {
    address_index: int
    amount: int
    token_index: int
}

TxTransferOutput {
    address: Address
    amount: int
    token_index: int
}
```

The fields mean:

- `addresses` carries the authorization material for debiting the global address-balance map
- `inputs` says how much value is debited from which authorized address
- `outputs` says how much value is credited to which global-balance address

`token_index` follows Hathor's existing token table semantics:

- index `0` means HTR
- index `n > 0` refers to `tx.tokens[n - 1]`

`address_index` points into `TransferHeader.addresses`.

`amount` fields are encoded with the same versioned output-value codec used by the other headers, i.e. `encode_output_value(serializer, amount, decimal_version=tx.decimal_version)`. Because this header only exists in the V2 era (see [Decimal precision](#decimal-precision-token-amount-v2)), **a `TransferHeader` is only valid on a V2 transaction** (`VertexDecimalVersion.V2`). A V1 transaction carrying a `TransferHeader` is invalid. Each `amount` is therefore an 18-decimal base-unit integer, bounded by the same maximum output value as regular outputs, `2^63 * 10^16` (Token Amount V2, I2).

The header is part of the transaction sighash. Therefore each `InputAddress.script` authorizes the full transaction, not only the transfer entries in isolation.

## Authorization and replay protection

Each `InputAddress` carries its own replay-protection state:

- `address`
- `seqnum`
- `script`

Validation requires:

- `script` must successfully authorize `address` under the same address-locking rules used by regular outputs
- `seqnum` must be strictly greater than `GlobalAddressSeqnum[address]`

On successful execution, the protocol sets:

```text
GlobalAddressSeqnum[address] = seqnum
```

This makes each debited address monotonic and replay-protected.

Gaps in seqnum are allowed: the protocol does not require `last_seqnum + 1`, only that the new seqnum is strictly greater than the stored value. However, a maximum jump size limits how far ahead a seqnum may be relative to the last committed value. This reuses the same constants defined for nano contract seqnums: block-level execution enforces `MAX_SEQNUM_JUMP_SIZE` (currently 10), and the mempool applies a wider window of `MAX_SEQNUM_DIFF_MEMPOOL` (currently 40). The bounded gap accommodates concurrent wallet implementations and transient network errors without allowing unbounded state divergence.

Standalone `TransferHeader` transactions may authorize multiple addresses. Each address has its own seqnum and signature.

For transactions that also carry a `NanoHeader`, this RFC is stricter whenever the transfer header debits the global map:

- if `TransferHeader.inputs` is non-empty, exactly one `InputAddress` is allowed
- that address must equal the caller address (`nc_address` in the nano header, which identifies the caller, not the contract being called)
- every `TxTransferInput` must reference `address_index = 0`

This RFC therefore allows caller-funded contract calls, but not arbitrary third-party debits hidden inside contract execution.

## Canonical transfer representation

`TransferHeader` is a net-state-change header, not a log of intermediate hops.

The header must satisfy all of the following:

- every `amount` is strictly positive
- every `InputAddress` is referenced by at least one `TxTransferInput`
- each `(address_index, token)` pair appears at most once in `inputs` (where `token` is the resolved token identity)
- each `(address, token)` pair appears at most once in `outputs`
- the same `(address, token)` pair may not appear on both sides of the header

If a transaction would both debit and credit the same `(address, token)` pair, it must encode only the net effect.

## Validation rules

This RFC defines the following validity rules for `TransferHeader`:

- only transactions may carry it
- the carrying transaction must be V2 (`tx.decimal_version == VertexDecimalVersion.V2`); a `TransferHeader` on a V1 transaction is rejected
- only one `TransferHeader` instance is allowed
- each `address_index` must point to an existing `InputAddress`
- each referenced `token_index` must resolve to HTR or a token in `tx.tokens`
- every debit address must be a valid regular address
- every credit address must be a valid regular address
- every `InputAddress.script` must successfully authorize its address
- every debited amount must be covered by the current `GlobalAddressBalance[(address, token)]`
- if the transaction also carries a `NanoHeader` and `TransferHeader.inputs` is non-empty, the only allowed debit address is the caller

`TransferHeader` is first-class. It is not limited to nano-contract transactions.

In particular:

- a transaction may have zero UTXO inputs if it is funded by transfer-header debits
- a transaction may have zero UTXO inputs and zero UTXO outputs if it settles entirely in the global balance map

## Accounting rule

`TransferHeader` does not introduce a separate accounting system. It extends Hathor's existing per-token balance equation.

For each token `T`, the transaction must satisfy:

```text
UTXOInputs[T]
+ TransferInputs[T]
+ NCWithdrawals[T]
=
UTXOOutputs[T]
+ TransferOutputs[T]
+ NCDeposits[T]
+ MintMeltAndFeeAdjustments[T]
```

Where:

- `TransferInputs[T]` is the sum of `TxTransferInput.amount` for token `T`
- `TransferOutputs[T]` is the sum of `TxTransferOutput.amount` for token `T`
- `NCWithdrawals[T]` and `NCDeposits[T]` are the existing nano-contract treasury movements
- `MintMeltAndFeeAdjustments[T]` covers existing token accounting rules such as mint authority, melt authority, and per-output fee calculations

Every term in this equation is evaluated in V2-normalized (18-decimal) units. The transaction is V2, so its transfer amounts, outputs, and nano actions are already V2; when it spends V1 UTXOs, those input values are scaled up by `10^16` before the equation is checked, so V2 values are never truncated to V1 (Token Amount V2, I6/I7). The equation itself is unchanged.

So, in balance verification:

- transfer inputs behave like additional transaction funding sources
- transfer outputs behave like additional transaction sinks

This is what makes all of the following valid under one accounting model:

- UTXO -> global balance deposits
- global balance -> UTXO withdrawals
- pure global balance transfers
- caller-funded nano-contract calls

## Ordering, mempool, and reorgs

Per-address seqnums create ordering dependencies between transactions from the same address. This is the same class of problem already solved for nano contract transactions. At the block level, execution uses a deterministic topological sort seeded by the block hash, which respects ordering constraints while introducing randomness to minimize MEV exploits. The mempool accepts transactions within a bounded seqnum window without requiring strict arrival order, since execution only happens when a block is produced.

Global address balances and seqnums are stored in the same patricia-trie structure used for nano contract state. Each block's metadata stores the trie root-id representing the full global state after that block's execution. During a reorg, the competing chain's state is computed by re-executing transactions on top of the common ancestor's root-id. Multiple competing states coexist in the trie without cross-contamination, so rollback is trivial — the "new" state is simply the result of applying the new executions on top of the parent block's root-id.

## Execution and state transition

For a standalone `TransferHeader` transaction, successful execution atomically:

- debits each referenced `GlobalAddressBalance[(address, token)]`
- credits each `TxTransferOutput`
- updates each consumed `GlobalAddressSeqnum[address]`

When a transaction carries both `TransferHeader` and `NanoHeader`, the whole transaction is one atomic state transition. Transfer-header debits and credits are applied before nano-contract method execution begins. This means:

- the contract sees the post-debit global address balances during execution
- transfer inputs contribute funds to the same per-token pool checked by transaction verification
- nano-contract execution may consume those funds through ordinary nano actions
- contract-local state changes, global-balance changes, and seqnum updates either all persist or none do

If contract execution fails, all changes are rolled back: no global-balance change or seqnum update is committed.

## Interaction with nano contracts

This RFC defines a minimal contract-facing interface to the global map:

```python
get_address_balance(address, token) -> amount
get_address_balance_before_current_call(address, token) -> amount
transfer_to_address(address, amount, token) -> None
```

Their semantics are:

- `get_address_balance(address, token)` reads `GlobalAddressBalance[(address, token)]` including any changes applied by the current transaction (transfer-header debits/credits and any prior `transfer_to_address` calls in this execution). This is consistent with `get_current_balance` for contract treasury balances.
- `get_address_balance_before_current_call(address, token)` reads `GlobalAddressBalance[(address, token)]` as it was before the current transaction began, ignoring any in-flight changes. This is consistent with `get_balance_before_current_call` for contract treasury balances.
- `transfer_to_address(address, amount, token)` debits the executing contract treasury and credits the target global address balance. If the contract's treasury has insufficient balance for the requested transfer, the call raises an execution failure, which causes the entire nano contract call to fail. Blueprint authors who want to avoid this should check the treasury balance before attempting a transfer.

These syscalls are available to **V2 blueprints only**. The global map stores 18-decimal values and V2 blueprints already operate in 18 decimals, so amounts cross the syscall boundary as raw V2 base-unit integers with no normalization. Calling them from a V1 blueprint is a runtime error, consistent with Token Amount V2's two-worlds model where V1 and V2 blueprints cannot interact (I15). This restriction is automatically satisfied for caller-funded calls: a `TransferHeader` implies a V2 transaction, which can only target V2 blueprints in the first place.

This RFC does not define a general-purpose syscall to debit arbitrary user balances from the global map.

That is intentional. Nano contracts execute arbitrary code, so user debits must remain explicit and transaction-authenticated. In this RFC, a contract may consume user value from the global map only when:

- the debited address is the caller
- the debit is declared in `TransferHeader`
- the caller's script authorizes it

So the contract can use caller-provided global-balance funds, but it cannot silently seize third-party balances.

## Token support

The global address-balance map is token-generic.

- HTR may be transferred through it
- existing custom tokens may be transferred through it
- pure global-balance transfers of custom tokens are valid as long as the token exists and appears in `tx.tokens` when required

`TransferHeader` does not create a second token namespace. Token identity and token-version checks continue to use Hathor's normal token metadata rules.

# Drawbacks
[drawbacks]: #drawbacks

This RFC adds a second protocol balance domain beside UTXOs. That increases complexity in several places:

- transaction validation
- wallet balance presentation
- explorer and indexer design
- API surface area

It also introduces per-address sequence state, which is more account-like than Hathor's current UTXO-only flows.

This feature also hard-depends on Token Amount V2: it cannot be used by V1 transactions or V1 blueprints, and its activation cannot precede V2's. Participants that have not migrated to V2 are excluded from the global balance map entirely.

Finally, the caller-only rule for nano-contract debits is intentionally conservative. It keeps authorization simple, but it does not cover every multi-party contract flow that might be desirable in the future.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why this design

This design gives Hathor a shared programmable balance space without discarding UTXOs.

That is the core tradeoff:

- keep UTXOs as the settlement layer
- add a protocol-owned address map for programmable user balances
- use one explicit header to bridge the two

This is narrower than a full account-model rewrite, but more interoperable than leaving every contract to invent its own ledger.

## Alternative: keep all programmable balances as UTXOs

This keeps the protocol simpler, but it preserves the current composition problem: transaction authors must know too much about the final UTXO shape before execution.

## Alternative: keep address maps inside each contract

This avoids a protocol change, but it fragments wallet support, explorer support, inter-contract conventions, and transfer semantics. The same user balance concept would be reimplemented many times with incompatible interfaces.

## Alternative: merge contract treasury and the global address-balance map

This RFC rejects that approach. Contract-owned funds and user-owned shared balances have different trust and authorization semantics. Keeping them distinct makes the debit rules clearer and avoids treating contract IDs as user addresses.

## Alternative: keep the current repeated-auth transfer-input shape

This RFC chooses an authorization list plus transfer entries instead.

That gives:

- one signature and seqnum per debited address
- no repeated script material for multiple tokens from the same address
- support for multi-address standalone transfers

It is a better fit than either repeating authorization on every input or forcing all transfer transactions into a single-source model.

## Alternative: let contracts debit arbitrary global balances

This RFC rejects that design. User debits should remain explicit, transaction-authenticated, and easy to audit. Arbitrary balance seizure hidden inside general contract code is too permissive for the first version of this feature.

# Prior art
[prior-art]: #prior-art

The closest prior art is the shared account-state model used by systems such as Ethereum, Solana, and Aptos, where programmable balances live in protocol state instead of explicit UTXOs.

This RFC differs by keeping UTXOs as Hathor's settlement format and adding a second balance domain beside them, rather than replacing them.

From the opposite direction, UTXO-based smart-contract systems show the cost of requiring every programmable balance movement to be expressed as explicit output planning. The more dynamic the computation, the more awkward that becomes.

This RFC takes a hybrid approach:

- UTXOs remain first-class
- global address balances add a shared programmable balance layer
- `TransferHeader` makes the bridge explicit

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should wallets present UTXO balance, global address balance, and contract-owned balances: separately, aggregated, or both?
- Which explorer and API endpoints should expose global address balances, and should they also expose seqnum state?
- Should a future RFC allow nano-contract transactions to consume multiple pre-authorized input addresses instead of only the caller?
- Should a future RFC expose more contract-facing operations over the global balance map beyond read and settle?
- What feature-activation schedule and rollout policy should be used for deployment on each network, given that activation must not precede Token Amount V2?
- Should a future RFC let V1 transactions and V1 blueprints participate in the global map through boundary normalization (scaling 2-decimal amounts by `10^16`, with lossy reads of sub-cent balances)? This RFC keeps the map V2-only and treats that as explicitly out of scope.
- What is the expected state growth profile for the global balance table, and should zero-balance entries be pruned from the trie to limit storage growth? Any pruning strategy must not affect consensus: currently, trie root-ids are not used for reaching consensus, so pruning is purely a storage optimization.
- How are transaction fees paid when a transaction has zero UTXO inputs? Can fees come from transfer-header debits, or must a minimal UTXO input always be present for fee payment?
- Should the `nc_address` field in `NanoHeader` be renamed or clarified to avoid confusion between the caller address and the contract address?

# Future possibilities
[future-possibilities]: #future-possibilities

If this model works well, the natural extensions are clear.

- richer wallet and explorer support for the shared balance layer
- APIs specialized for global address balances
- more expressive multi-party authorization flows for contract calls
- additional contract APIs that build on the same authorization model without collapsing user balances and contract treasury into one space

- pruning zero-balance entries from the global address-balance trie as a storage optimization

The intended long-term direction is not to replace UTXOs, but to make global address balances the standard shared programmable balance layer for user-owned funds on Hathor.
