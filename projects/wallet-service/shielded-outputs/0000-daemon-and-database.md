- Feature Name: `wallet_service_shielded_outputs_daemon_schema`
- Start Date: 2026-05-12
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Add shielded-output support to the wallet-service by **extending the existing `tx_output`, `address`, `address_balance`, `wallet_balance`, `address_tx_history`, and `wallet_tx_history` tables**. A new `mode` column on `tx_output` discriminates `0 = transparent`, `1 = AMOUNT_SHIELDED`, `2 = FULLY_SHIELDED`. The heavy shielded-only on-chain bytes (commitment, range proof, ephemeral pubkey, script, plus mode-conditional fields) live in a 1:1 satellite table `shielded_tx_output_data` keyed by the same `(tx_id, index)` PK as `tx_output`. Shielded ownership folds into the existing `address` table via a new `bip32_account` discriminator column (`0 = transparent`, `1 = shielded scan path`); rows continue to exist for every observed spend_address, and the shielded scan key, long-form display address, and catchup state live on the same row.

The daemon's `handleVertexAccepted`, `handleVoidedTx`, `handleUnvoidedTx`, `handleVertexRemoved`, and input-consumption code paths gain shielded-aware logic but are **not duplicated**: there is no `voidShieldedOutputs`, no `processShieldedOutputs` outside the main loop, no parallel reorg pipeline. The only kind-specific work is balance-column dispatch on `tx_output.mode` inside `updateAddressTablesWithTx`.

# Motivation
[motivation]: #motivation

Hathor is introducing **shielded outputs** — a new output type that hides amount (and optionally token) using Pedersen commitments, Bulletproofs, and surjection proofs. The wallet-service is the canonical indexer for hosted Hathor wallets: it must index shielded outputs the same way it indexes transparent outputs, while keeping the existing transparent code path stable.

Reorg, void, unvoid, and input-consumption are the most sensitive parts of the daemon today, and reorg correctness is already exercised by integration tests for the transparent path. Building a parallel shielded subsystem alongside the existing one would duplicate that surface; every reorg, void, and unvoid handler would gain a near-textual shielded twin that must stay correct in lockstep with its transparent counterpart.

This design takes a different trade-off: **modify the hot tables and the hot handlers to be kind-aware**, so reorg/void/unvoid logic is written once. The cost is that:

- `tx_output.value` and `tx_output.token_id` become nullable.
- Rows in `tx_output` carry an additional discriminator (`mode`) that every kind-sensitive query must respect.
- `updateAddressTablesWithTx` and `updateWalletTablesWithTx` gain a switch on `mode` to update the right balance columns.

The benefit is:

- Reorg, void, unvoid, vertex-removal, and input-consumption code paths operate on `tx_output` rows uniformly. No parallel handler to maintain in lockstep.
- A single indexed lookup against `tx_output` resolves any input `(prev_tx_id, prev_index)`, regardless of whether the consumed output was transparent or shielded.
- The application-layer "read the wallet's UTXO set" helper is one query (with an optional `kind` filter), not a UNION of two tables.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A wallet-service contributor working on the daemon today thinks in terms of one ingestion pipeline: a vertex arrives, `handleVertexAccepted` walks `outputs[]`, and the result is rows in `tx_output`, `address_balance`, and `wallet_balance`.

After this RFC, **the same one pipeline does the work**, but the loop iterates over both `outputs[]` and `shielded_outputs[]` and tags every `tx_output` row it writes with a `mode` column:

```
                      handleVertexAccepted (vertex v)
                                 │
                                 ▼
   for each output in v.outputs[] ++ v.shielded_outputs[]:
     insert tx_output row with mode = 0 | 1 | 2
     if mode in (1, 2):
       insert shielded_tx_output_data row with crypto bytes
       upsert address row with bip32_account = 1 (create if new with NULL ownership; long-form shielded_address stays NULL until wallet registration populates it)
     if output is owned:
       update address_balance, wallet_balance, *_tx_history
       (column dispatch on mode)
```

Everything runs inside the existing per-vertex DB transaction.

The contributor must hold four concepts:

1. **`tx_output.mode`** — the single discriminator. `0 = transparent`, `1 = AMOUNT_SHIELDED`, `2 = FULLY_SHIELDED`. Every kind-sensitive query filters on this column. Existing queries that omit it are implicitly transparent-only **only if explicitly filtered**, which is why a migration step adds `WHERE mode = 0` to every existing `tx_output` query that should remain transparent-scoped.

2. **`recovery_state`** — `tx_output.recovery_state` (NULL for `mode = 0`; `unowned` / `recovered` / `recovery_failed` for `mode IN (1, 2)`). For unowned shielded outputs, `value` is always NULL; for `mode = 1` the `token_id` is populated at observe time from `token_data` (visible on the wire), so only `value` is filled by recovery; for `mode = 2`, both `value` and `token_id` are NULL until recovery.

3. **Eager rewind on match.** When a shielded output's `address` matches an `address` row with `bip32_account = 1` and non-NULL `wallet_id`, the daemon synchronously calls `rewindAmountShieldedOutput` or `rewindFullShieldedOutput` (from `@hathor/ct-crypto-node`) using the cached per-index `scan_privkey` for that row. On success, the recovered `{ value, token_id }` is written back to the **same** `tx_output` row (`recovery_state = 'recovered'`). Blinding factors are not stored.

4. **Single-table reorg/void.** The void path sets `voided = TRUE` on `tx_output` rows. The balance reversal calls the same `updateAddressTablesWithTx` (with a reversing sign) that the apply path used; that function switches on `mode` to reverse the right balance columns. **There is no separate shielded handler.**

A contributor adding a new feature that needs to "iterate over a wallet's UTXOs" reads `tx_output` once, optionally filtered by `mode`. The wallet-service helper layer (`packages/wallet-service/src/db/`) exposes `getUtxos(walletId, { kind })` where `kind` defaults to "both".

## Worked example: receive

User Alice's wallet has shielded keys registered. Bob sends Alice 1.5 HTR via an `AMOUNT_SHIELDED` output.

1. Fullnode emits `NEW_VERTEX_ACCEPTED` for the transaction. The vertex event has a top-level `shielded_outputs[]` array with one entry.
2. Daemon's `handleVertexAccepted` runs. It iterates `outputs[]` (empty in this example) and then `shielded_outputs[]` (one entry).
3. For the shielded entry, it INSERTs a `tx_output` row with:
   - `mode = 1`
   - `address = decoded.address` (base58 spend address)
   - `index = 0` (concatenated index)
   - `value = NULL`
   - `token_id = '00'` (from `token_data`)
   - `recovery_state = 'unowned'`
4. It INSERTs a `shielded_tx_output_data` row with the crypto bytes.
5. It UPSERTs `address`: row for Alice's `decoded.address` already exists with `bip32_account = 1`, `wallet_id = wallet_alice`, `index = 7`, `scan_privkey = …`, `shielded_address = <long-form display string>` because Alice's wallet-registration step pre-derived this row earlier (registration is where the long-form is computed from `scan_privkey` + `spend_pubkey` and written to the row — see [[0001-wallet-registration]]). The single canonical `transactions` bump for `(address, tx)` happens once per vertex inside `updateAddressTablesWithTx` (see § *Invariants*); the observation upsert itself does not touch `transactions`.
6. It sees `wallet_id IS NOT NULL` on the looked-up row → owned. It calls `rewindAmountShieldedOutput(scan_privkey, ephemeral_pubkey, commitment, range_proof, token_uid_from_token_data)`. Returns `{ value: 150 }`.
7. It UPDATEs the `tx_output` row: `value = 150`, `recovery_state = 'recovered'`. (`token_id` is already populated.)
8. It calls `updateAddressTablesWithTx`, which dispatches on `mode = 1`:
   - Bump `address_balance.unlocked_shielded_balance += 150` and `total_shielded_received += 150` for `(Alice_spend_address, '00')`. (The same row also carries transparent columns; those stay zero here.)
   - Bump `wallet_balance.unlocked_shielded_balance += 150` for `(wallet_alice, '00')`.
   - Append `wallet_tx_history` row with `shielded_balance_delta = 150` (and `balance_delta = 0`); append `address_tx_history` row with `shielded_balance_delta = 150` for `(Alice_spend_address, tx_id, '00')`.
9. It enqueues a `'new-tx'` event carrying both transparent (empty here) and shielded deltas.
10. The DB transaction commits.

If the same vertex also had transparent outputs, those would be processed in the same loop and contribute to the **same** `wallet_tx_history` row (one row per `(wallet_id, tx_id, token_id)`), with `balance_delta` carrying the transparent delta and `shielded_balance_delta` carrying the shielded delta.

## Worked example: mixed receive (transparent + shielded in one tx)

Bob sends Alice 1.0 HTR via a transparent output **and** 1.5 HTR via an `AMOUNT_SHIELDED` output in the same transaction. Alice's wallet has shielded keys registered.

1. Fullnode emits `NEW_VERTEX_ACCEPTED`. The vertex has `outputs[]` (one entry, 1.0 HTR transparent to Alice) and `shielded_outputs[]` (one entry, 1.5 HTR shielded to Alice).
2. `handleVertexAccepted` builds the concatenated list and iterates:
   - **`outputs[0]` (transparent, concatenated index 0).** INSERTs `tx_output` row with `mode = 0`, `address = Alice_transparent_address`, `value = 100`, `token_id = '00'`, `recovery_state = NULL`. The existing transparent ownership lookup against `address` (with `bip32_account = 0`) finds `wallet_id = wallet_alice`.
   - **`shielded_outputs[0]` (mode=1, concatenated index 1).** INSERTs `tx_output` row with `mode = 1`, `address = Alice_spend_address`, `value = NULL`, `token_id = '00'` (from `token_data`), `recovery_state = 'unowned'`. INSERTs satellite row with crypto bytes. UPSERTs the `address` row for `Alice_spend_address` (already owned by `wallet_alice` at `bip32_account = 1`, `index = 7`). Rewinds → `value = 150`. UPDATEs the `tx_output` row to `value = 150`, `recovery_state = 'recovered'`.
3. `updateAddressTablesWithTx` runs over both rows in the same call, dispatching on `mode`:
   - `address_balance` for `(Alice_transparent_address, '00')`: bump `unlocked_balance += 100`, `total_received += 100`.
   - `address_balance` for `(Alice_spend_address, '00')`: bump `unlocked_shielded_balance += 150`, `total_shielded_received += 150`. (Same row layout as the transparent row above; only the shielded columns move.)
4. `updateWalletTablesWithTx` aggregates per-token across both kinds:
   - `wallet_balance` for `(wallet_alice, '00')`: `unlocked_balance += 100`, `unlocked_shielded_balance += 150`, `transactions += 1`.
   - `wallet_tx_history`: **one row** for `(wallet_alice, tx_id, '00')` with `balance_delta = 100` and `shielded_balance_delta = 150`.
5. Emits a single `'new-tx'` event with `balance_changes_by_token: [{ token_id: '00', balance_delta: 100, shielded_balance_delta: 150 }]`.
6. The DB transaction commits.

Note: the `wallet_tx_history` row is one row, not two. The API derives `output_kind = 'mixed'` from "both deltas non-zero" when it renders this entry.

## Worked example: send a shielded UTXO with shielded change

Alice spends a 4.0 HTR `AMOUNT_SHIELDED` UTXO to send 1.5 HTR shielded to Bob and 2.5 HTR shielded back to herself as change. Bob is not tracked by this wallet-service.

The vertex looks like:

- `inputs[0]`: `(prev_tx_id = T0, prev_index = 5)`, `spent_output = { mode: 1, decoded.address: Alice_spend_address, … }` (the full prior shielded output is on the wire).
- `outputs[]`: empty.
- `shielded_outputs[]`: two entries — `[0]` to Bob (1.5 HTR), `[1]` to Alice's change address (2.5 HTR).

1. `handleVertexAccepted` iterates outputs first:
   - **`shielded_outputs[0]` (Bob's, concatenated index 0).** INSERTs `tx_output` row with `mode = 1`, `address = Bob_spend_address`, `value = NULL`, `token_id = '00'`, `recovery_state = 'unowned'`. INSERTs satellite. UPSERTs the `address` row for `Bob_spend_address` — row didn't exist before (this is a brand-new observation), so an INSERT runs with `bip32_account = 1`, `wallet_id = NULL`, `shielded_address = NULL` (the long-form cannot be derived from the on-chain spend address alone — it requires `scan_pubkey` + `spend_pubkey`, which the row only gains when a wallet later registers and pre-derives them). Ownership lookup: `wallet_id IS NULL` → unowned, no rewind, no balance update.
   - **`shielded_outputs[1]` (Alice's change, concatenated index 1).** INSERTs `tx_output` with same shape as the prior example. UPSERTs the `address` row (Alice's change address is owned at `bip32_account = 1`, `index = 12`, say). Rewinds → `value = 250`. UPDATEs `tx_output` to `value = 250`, `recovery_state = 'recovered'`.
2. Then the input loop:
   - **`inputs[0]`.** `markTxOutputSpent(tx_id=T0, index=5)` runs `UPDATE tx_output SET spent_by = <current_tx_id> WHERE tx_id = T0 AND index = 5`. The kind of the consumed output is read from `input.spent_output.mode = 1` directly — no second query.
3. `updateAddressTablesWithTx` and `updateWalletTablesWithTx` aggregate every row touched by the vertex:
   - From the spent input: `-400` shielded delta (Alice's old 4.0 HTR UTXO is now consumed).
   - From the new owned output `shielded_outputs[1]`: `+250` shielded delta (Alice's change).
   - From the new unowned output `shielded_outputs[0]`: no balance impact (unowned).
4. Resulting writes:
   - `address_balance` for `(Alice_spend_address_of_T0_index_5, '00')`: `unlocked_shielded_balance -= 400` (reversal helper UPDATEs only the `*_balance` columns; `total_shielded_received` is not touched because this is a spend reversal, not a void of the original receive — the T0 receive that credited `total_shielded_received` is still valid).
   - `address_balance` for `(Alice_change_spend_address, '00')`: `unlocked_shielded_balance += 250`, `total_shielded_received += 250`.
   - `wallet_balance` for `(wallet_alice, '00')`: `unlocked_shielded_balance += (-400 + 250) = -150`.
   - `wallet_tx_history`: one row with `balance_delta = 0`, `shielded_balance_delta = -150`. `address_tx_history` carries the same `shielded_balance_delta` on the per-address rows.
5. Bob's wallet-service (if he uses one) sees the same vertex independently and runs the same flow against its own `address` table; for him, Bob's output is owned and recovered, and Alice's change is unowned.

The point of this example: **the input's `spent_output.mode` is the only signal needed to dispatch the balance reversal**, and the `UPDATE tx_output SET spent_by = …` works on shielded rows with no special case.

## Worked example: third-party shielded send (nobody on this wallet-service is involved)

Charlie sends Dave a shielded output. Neither Charlie nor Dave has a wallet registered on this wallet-service. The daemon still observes and persists the data — this is how catch-up works when Dave (or Charlie) later registers.

1. `handleVertexAccepted` iterates `shielded_outputs[]`:
   - INSERTs `tx_output` with `mode = 1`, `address = Dave_spend_address`, `value = NULL`, `token_id = '00'`, `recovery_state = 'unowned'`.
   - INSERTs satellite row with crypto bytes.
   - UPSERTs the `address` row for `Dave_spend_address` (new row, `bip32_account = 1`, `wallet_id = NULL`, `shielded_address = NULL` — the long-form is populated only when Dave's wallet later registers and derives it from `scan_privkey` + `spend_pubkey`).
   - Ownership lookup: `wallet_id IS NULL` → no rewind, no balance update.
2. The input loop processes Charlie's spent input the same way: `UPDATE tx_output SET spent_by = …` (Charlie's prior UTXO was observed by the daemon the same way, with NULL ownership).
3. `updateAddressTablesWithTx` runs and finds nothing to do (no `wallet_id` owns any of the touched addresses on this service). No `address_balance` / `wallet_balance` writes.
4. No `'new-tx'` event is emitted to any wallet (no subscriber).
5. The DB transaction commits.

A week later, Dave registers shielded keys. His registration UPSERTs `address` rows (`bip32_account = 1`) for his first 20 shielded indexes; one of those addresses **is** the row already created at step 1 above, so the UPSERT fills in `wallet_id`, `index`, `scan_privkey`, `catchup_state`, etc., transitioning the row from "observed, unowned" to "owned, catchup pending". The catch-up job then scans `tx_output WHERE mode IN (1,2) AND recovery_state = 'unowned' AND address IN (Dave's claimed set)`, finds the row written at step 1, runs the rewind, and credits Dave's balance retroactively.

## Worked example: FULLY_SHIELDED receive

Alice receives 0.75 of token `T1` via a `FULLY_SHIELDED` output (mode = 2). The token type is also hidden on the wire.

1. `handleVertexAccepted` iterates `shielded_outputs[]`:
   - INSERTs `tx_output` with `mode = 2`, `address = Alice_spend_address`, `value = NULL`, `token_id = NULL` (token is hidden — no `token_data` to read), `recovery_state = 'unowned'`.
   - INSERTs satellite row including `asset_commitment` and `surjection_proof` (the mode-2-only fields).
   - UPSERTs the `address` row for Alice's spend address (owned, `bip32_account = 1`).
   - Ownership lookup hits. Calls `rewindFullShieldedOutput(scan_privkey, ephemeral_pubkey, commitment, range_proof, asset_commitment)`. Returns `{ value: 75, tokenUid: 'T1', assetBlindingFactor: … }`.
   - Verifies the asset commitment against the recovered token (`verifyAssetCommitment`) — a mandatory step for mode 2.
   - UPDATEs `tx_output`: `value = 75`, `token_id = 'T1'`, `recovery_state = 'recovered'`.
2. `updateAddressTablesWithTx` updates `address_balance` for `(Alice_spend_address, 'T1')`: `unlocked_shielded_balance += 75`, `total_shielded_received += 75`.
3. `wallet_balance` for `(wallet_alice, 'T1')`: `unlocked_shielded_balance += 75`. If this is the first time wallet_alice has seen token T1, the row is created by the upsert.
4. `wallet_tx_history` row written with `shielded_balance_delta = 75` for `(wallet_alice, tx_id, 'T1')`. `address_tx_history` carries the same `shielded_balance_delta = 75` on the per-address row.

The point of this example: `token_id` lives in `tx_output` and is filled in by recovery for mode 2, whereas for mode 1 it would have been filled at observe time. The handler dispatches on `mode` to know which.

## Worked example: void of a recovered shielded receive

Two days after the first worked example (Alice received 1.5 HTR shielded), the transaction is reorged out and gets voided.

1. `handleVoidedTx` runs for the vertex.
2. UPDATEs every `tx_output` row of the vertex: `voided = TRUE` (affects both transparent and shielded rows uniformly).
3. UPDATEs `wallet_tx_history` and `address_tx_history` rows for the same `tx_id`: `voided = TRUE`.
4. Reverses the balance deltas by re-running `updateAddressTablesWithTx` / `updateWalletTablesWithTx` with a reversing sign. For each `tx_output` row of the vertex:
   - The row carries `mode = 1` and `recovery_state = 'recovered'` (it was recovered on receive).
   - Dispatch on `mode` picks the shielded balance columns. `wallet_balance.unlocked_shielded_balance -= 150` and `total_shielded_received -= 150` for `(wallet_alice, '00')`. `address_balance.unlocked_shielded_balance -= 150` and `total_shielded_received -= 150` for `(Alice_spend_address, '00')` on the unified row (no `kind` filter; the reversal helper UPDATEs the shielded columns and decrements `total_shielded_received` because this is a void of the original receive, mirroring how `voidAddressTransaction` decrements `total_received` for a voided transparent receive).
5. Push notification or WebSocket event emitted to wallet_alice indicating the void.

Crucially: this is the **same** `handleVoidedTx` that runs for a transparent void; the only kind-specific work is the column dispatch inside `updateAddressTablesWithTx`. There is no `voidShieldedOutputs` companion handler. The same observation applies to `handleUnvoidedTx` (re-applies) and `handleVertexRemoved` (delete, with FK cascading the satellite row).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Invariants

These hold at every commit and constrain how the per-vertex pipeline writes the address-grain and wallet-grain tables. Every helper below is structured to maintain them.

- **Two grains of `transactions` counter, two single-writer invariants.**
  - *Involvement counter* — `address.transactions`. Bumped exactly once per `(address, tx)`, driven by the involved-addresses set computed from wire data (inputs' `spent_output.decoded.address` + transparent outputs' `decoded.address` + shielded outputs' `decoded.address`). The bump fires regardless of whether the vertex carries a known token or value for that address, so unowned-shielded observations, recovery-failed shielded receives, and any other "we saw this address but don't know what happened" cases all contribute exactly one bump. The single writer is the address-grain bumper called inside `updateAddressTablesWithTx` (or a companion helper run at the same point in `handleVertexAccepted`). The observation upsert (`upsertShieldedAddressObservation`) does **not** bump.
  - *Per-token counter* — `address_balance.transactions` and `wallet_balance.transactions`. Bumped exactly once per `(address, token, tx)` and per `(wallet, token, tx)`, driven by the unified per-token balance map. The map only contains entries where both token and value are known (transparent inputs/outputs, recovered+owned shielded inputs/outputs); unrecovered or unowned shielded entries are excluded, so no phantom `(address, '')` or `(wallet, '')` row can appear. The single writer is `updateAddressTablesWithTx` / `updateWalletTablesWithTx`.
  - No wallet-grain involvement counter exists. `wallet.transactions` is intentionally not a column — wallet involvement is implicit in the ownership of involved addresses; consumers that need it can `JOIN` through `address.wallet_id`.
- **Lifetime totals reflect currently-valid receipts.** `total_received` and `total_shielded_received` (on `address_balance` and `wallet_balance`) are incremented by credit helpers when a receive is observed and decremented by reversal helpers when the receive is later voided — matching the transparent path's existing behavior (`voidAddressTransaction` subtracts `totalAmountSent` from `total_received` today). A voided receive is removed from the lifetime total so consumers see the lifetime amount that was actually delivered, not the gross including rolled-back txs. For shielded, the decrement only fires for outputs whose value was known when the credit happened (`recovery_state = 'recovered'`); unrecovered shielded outputs never contributed to `total_shielded_received` in the first place, so their void is a no-op for this column.
- **`address.bip32_account` is `NOT NULL DEFAULT 0`.** The migration backfills existing transparent rows to `0` in the same `ADD COLUMN` statement, so the unique constraint `(wallet_id, bip32_account, index)` enforces strictly from the moment it lands. The other three new columns (`scan_privkey`, `catchup_state`, `shielded_address`) stay nullable — populated only for `bip32_account = 1` rows.

## Schema changes

### `tx_output` — modified

```
tx_output
  -- existing columns (some widened — see migration plan)
  tx_id                   VARCHAR(64)         NOT NULL,
  index                   SMALLINT UNSIGNED   NOT NULL,  -- WIDENED from TINYINT UNSIGNED; see migration plan. Concatenated index (outputs ++ shielded_outputs).
  address                 VARCHAR(34)         NOT NULL,  -- decoded.address for both transparent and shielded
  value                   BIGINT UNSIGNED NULL,      -- CHANGED: now nullable (NULL for unrecovered shielded)
  token_id                VARCHAR(64)     NULL,      -- CHANGED: now nullable (NULL for unrecovered FULLY_SHIELDED)
  authorities             SMALLINT        NOT NULL DEFAULT 0,  -- always 0 for mode IN (1, 2)
  timelock                INT UNSIGNED    NULL,
  heightlock              INT UNSIGNED    NULL,
  locked                  BOOLEAN         NOT NULL DEFAULT FALSE,
  spent_by                VARCHAR(64)     NULL,
  voided                  BOOLEAN         NOT NULL DEFAULT FALSE,
  first_block             VARCHAR(64)     NULL,
  -- new columns
  mode                    TINYINT         NOT NULL DEFAULT 0,  -- 0 = transparent, 1 = AMOUNT_SHIELDED, 2 = FULLY_SHIELDED
  recovery_state          ENUM('unowned','recovered','recovery_failed') NULL,  -- NULL when mode = 0
  PRIMARY KEY (tx_id, index),
  INDEX idx_address (address),
  INDEX idx_mode_recovery (mode, recovery_state),
  INDEX idx_spent_by (spent_by),
  INDEX idx_voided_mode (voided, mode)
```

Notes:

- `mode DEFAULT 0` means the migration that adds the column leaves every existing row as transparent — correct because every existing row is transparent.
- `value` and `token_id` are nullable so the row can exist between observation and recovery for shielded outputs.
  - For `mode = 0`: both populated.
  - For `mode = 1`: `token_id` populated at observe time (from `token_data`); `value` NULL until recovery.
  - For `mode = 2`: both NULL until recovery.
- `recovery_state` is NULL for transparent rows. For shielded rows it transitions `unowned → recovered` on a successful rewind, or `unowned → recovery_failed` on a rewind error.
- `authorities` stays on `tx_output` because authority bits are a transparent-only concept; on shielded rows it is always 0. (The upstream RFC keeps authority outputs transparent; an in-flight follow-on adds shielded *value* outputs to mint/melt transactions while keeping the authority outputs themselves transparent — see § *Future possibilities*.)
- `idx_mode_recovery` accelerates the catch-up scan (`WHERE mode IN (1,2) AND recovery_state = 'unowned' AND address IN (...)`).
- `idx_voided_mode` accelerates voided-row sweeps if any kind-specific cleanup runs.

#### Existing-query migration plan

Every existing query against `tx_output` that should remain transparent-only gains an explicit `WHERE mode = 0` clause as part of this migration. The migration script greps for `FROM tx_output` and lists the call sites; the implementation PR walks them. Queries that should naturally span both kinds (e.g., the `(prev_tx_id, prev_index)` input-resolution lookup) omit the filter intentionally.

#### Type-widening migration for `tx_output.index`

The existing `tx_output.index` column is `TINYINT UNSIGNED` (range 0–255), sized for the transparent protocol's hard cap of 255 outputs per transaction. With shielded outputs concatenated after transparent ones (up to 15 shielded entries appended to up to 255 transparent ones), the maximum concatenated index now exceeds 255 and overflows the current column. The migration widens it:

```sql
ALTER TABLE `tx_output` MODIFY COLUMN `index` SMALLINT UNSIGNED NOT NULL;
```

SMALLINT UNSIGNED (0–65535) is far more than needed (max ~270) but mirrors the existing UNSIGNED convention and is the smallest standard MySQL integer width that fits. Downstream code that types this column as a JS `number` is unaffected; any code that explicitly types it as `uint8_t` / `≤ 255` needs auditing. The same widening applies to the wire format's `prev_index` on inputs — if the fullnode emits `prev_index` as a single byte today, the shielded-output protocol upgrade is widening it server-side; the daemon parser accepts the wider integer naturally because Zod's `z.number().int()` carries no upper bound.

### `shielded_tx_output_data` — new, 1:1 satellite

Holds the heavy on-chain cryptographic bytes for every shielded `tx_output` row. Keyed by the same composite PK.

```
shielded_tx_output_data
  tx_id              VARCHAR(64)         NOT NULL,
  `index`            SMALLINT UNSIGNED   NOT NULL,
  commitment         VARBINARY(33)       NOT NULL,
  range_proof        VARBINARY(1024)     NOT NULL,
  script             VARBINARY(1024)     NOT NULL,  -- raw, not parsed
  ephemeral_pubkey   VARBINARY(33)       NOT NULL,
  token_data         TINYINT UNSIGNED    NULL,      -- non-null iff mode = 1; 1-byte wire value (low 7 bits = token index into vertex.tokens[], high bit = authority flag — always 0 for shielded)
  asset_commitment   VARBINARY(33)       NULL,      -- non-null iff mode = 2
  surjection_proof   VARBINARY(4096)     NULL,      -- non-null iff mode = 2
  PRIMARY KEY (tx_id, `index`),
  FOREIGN KEY (tx_id, `index`) REFERENCES tx_output(tx_id, `index`) ON DELETE CASCADE
```

The `index` column is `SMALLINT UNSIGNED` to match the widened `tx_output.index` (see § *Type-widening migration for `tx_output.index`*) and backticked because `index` is a SQL-reserved keyword in MySQL. Using the same column name and type as the parent table makes the FK self-documenting and removes the `output_index` ↔ `index` naming asymmetry an earlier revision had. `token_data` is `TINYINT UNSIGNED` to cover the full 0–255 byte range the wire emits — signed TINYINT (-128..127) would not fit `token_data` values ≥ 128.

Notes:

- The FK with `ON DELETE CASCADE` guarantees `handleVertexRemoved`'s `DELETE FROM tx_output WHERE tx_id = ?` automatically cleans up the satellite — no application-layer second delete.
- `script` is stored raw for archival. The daemon does not parse it.
- `recovery_state` is **not** here — it lives on `tx_output` so consumers can filter by it without joining the satellite. This is the answer to "where do I look to find every owned-but-unrecovered output for catch-up": `SELECT … FROM tx_output WHERE mode IN (1,2) AND recovery_state = 'unowned' AND address IN (…)`.

### `address` — modified (shielded columns added)

Shielded ownership folds into the existing transparent `address` table; there is **no** separate `shielded_address` table. A new `bip32_account` discriminator column (`0 = transparent`, `1 = shielded scan path`) tells the two kinds of rows apart on the same physical table. The existing PK `(address)` stays — the on-chain base58 spend address is unique within a kind, and the design assumes the two address spaces are disjoint (a given on-chain string is observed in only one role on this service).

```
address
  -- existing columns
  address           VARCHAR(34)    NOT NULL,
  wallet_id         VARCHAR(64)    NULL,
  `index`           INT UNSIGNED   NULL,
  transactions      INT UNSIGNED   NOT NULL DEFAULT 0,
  -- new columns
  bip32_account     TINYINT UNSIGNED NOT NULL DEFAULT 0,   -- 0 = transparent, 1 = shielded scan path
  scan_privkey      VARBINARY(32)  NULL,                    -- plaintext; NULL for transparent rows
  catchup_state     ENUM('pending','running','done') NULL,  -- NULL for transparent or unowned rows
  shielded_address  VARCHAR(100)   NULL,                    -- long-form display string; 71-byte payload, base58, ≤100 chars; NULL for transparent rows AND for unowned shielded rows (populated only at wallet registration, since the long-form requires scan_pubkey + spend_pubkey)
  PRIMARY KEY (address),
  UNIQUE KEY uk_address_wallet_account_index (wallet_id, bip32_account, `index`),
  INDEX idx_address_shielded_long (shielded_address),
  INDEX idx_address_wallet_catchup (wallet_id, catchup_state)
```

Notes:

- `bip32_account` is `NOT NULL DEFAULT 0`. The `ADD COLUMN` migration backfills every existing transparent row to `0` in the same statement, so the new unique key `(wallet_id, bip32_account, index)` enforces strictly from the moment it lands. Shielded rows are written with `bip32_account = 1`.
- The other three new columns (`scan_privkey`, `catchup_state`, `shielded_address`) stay nullable — they are only populated for `bip32_account = 1` rows.
- Daemon ingestion writes a row on every observed shielded spend address (mirroring how transparent `address` rows are written on observation): the upsert sets `bip32_account = 1` and creates the row if it doesn't exist. It does **not** write `shielded_address`, `scan_privkey`, or `catchup_state` — those columns are owned by the wallet-registration path, which has the material to derive the long-form (`scan_privkey` + `spend_pubkey`) and persists it in the same UPSERT that claims ownership. For an already-claimed wallet's row, the observation upsert is a no-op on every ownership column; it just confirms `bip32_account = 1` and exits.
- The single canonical `transactions` bump per `(address, tx)` lives in `updateAddressTablesWithTx` (see § *Invariants* below). The observation upsert does **not** bump it.
- `scan_pubkey` and `spend_pubkey` are **not** stored — they are intermediate values during derivation: `scan_pubkey` is recoverable via point multiplication from `scan_privkey`, and `spend_pubkey` is recoverable via BIP32 derivation from `wallet.spend_xpub` at the row's `index`.
- `scan_privkey` uses raw `ALTER TABLE … ADD COLUMN scan_privkey VARBINARY(32)` rather than the Sequelize BLOB family, because Sequelize's BLOB types map to `(TINY|MEDIUM|LONG)BLOB` and don't expose a fixed-cap VARBINARY variant. `scan_privkey` is always exactly 32 bytes (Ristretto255 scalar); VARBINARY is preferred over BLOB for inline storage and tight upper-bound typing.

### Why one address table with a `bip32_account` discriminator?

An earlier revision of this design used a separate `shielded_address` table. It was abandoned once it became clear that the shielded `spend_address` is a P2PKH derived on a separate BIP32 account (`m/44'/280'/1'`) and therefore shares the same on-chain shape as a transparent address — and can in principle receive both shielded *and* transparent outputs. Folding into one table eliminates a parallel ownership lookup path (`findShieldedAddressOwnership` becomes a filter on `bip32_account = 1` of the same SELECT) and removes a class of subtle bugs where the shielded ownership table and the transparent one drift out of sync during reorgs.

The trade-off is that the row layout grows four nullable columns whose values are only meaningful for `bip32_account = 1`. That is a small price for collapsing two parallel handlers into one.

### `address_balance` — modified

```
address_balance
  -- existing columns
  address                    VARCHAR(34)     NOT NULL,
  token_id                   VARCHAR(64)     NOT NULL,
  unlocked_balance           BIGINT UNSIGNED NOT NULL DEFAULT 0,
  locked_balance             BIGINT UNSIGNED NOT NULL DEFAULT 0,
  timelock_expires           INT UNSIGNED    NULL,
  transactions               INT UNSIGNED    NOT NULL DEFAULT 0,
  total_received             BIGINT UNSIGNED NOT NULL DEFAULT 0,
  -- existing authority columns unchanged
  -- new shielded columns
  unlocked_shielded_balance  BIGINT UNSIGNED NOT NULL DEFAULT 0,
  locked_shielded_balance    BIGINT UNSIGNED NOT NULL DEFAULT 0,
  total_shielded_received    BIGINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (address, token_id)
```

Notes:

- The same `(address, token_id)` row carries both transparent and shielded balances. There is no `kind` discriminator — the layout mirrors `wallet_balance`, with separate column pairs for the two kinds of balance.
- This shape exists because the shielded spend_address is a P2PKH that can in principle receive both transparent *and* shielded outputs. The same row then holds both kinds; consumers read one row and pick the columns they care about.
- **Column-sign rationale.** All balance and total columns — transparent and shielded alike — are `BIGINT UNSIGNED`, matching the existing transparent convention on master. The invariant is that every reversal subtracts a value previously added in the same column, so columns stay ≥ 0 under correct operation; underflow would indicate a daemon bug. Spend-reversal helpers use UPDATE-only SQL (`UPDATE … SET col = col - ? WHERE …`) rather than INSERT…ON DUPLICATE KEY UPDATE with negative `VALUES()`, because MySQL's STRICT_TRANS_TABLES (the default in 8.0) validates the literal in `VALUES(...)` against the column type *before* picking the update branch and rejects negative values against UNSIGNED columns. `total_*_received` is also `BIGINT UNSIGNED` and decremented on void (not on spend) — see next point for the semantic.
- `total_received` and `total_shielded_received` track currently-valid lifetime receipts. Credits add to them on receive; reversals subtract on void (matching today's `voidAddressTransaction` behavior on the transparent column). Spend reversals do **not** touch the `total_*_received` columns — only voids do, because only a void rolls back the receive itself. For shielded, the decrement only fires when the voided output had `recovery_state = 'recovered'` (its value was known at credit time); unrecovered shielded outputs never contributed and stay at zero impact on void.
- `transactions` is a single combined counter on the row, bumped at most once per `(address, tx)` regardless of kind mix. The bump lives in `updateAddressTablesWithTx`.
- Authority bits stay populated for the transparent columns; shielded outputs cannot carry authority.

### `address_tx_history` — modified

```
address_tx_history
  address                  VARCHAR(34)     NOT NULL,
  tx_id                    VARCHAR(64)     NOT NULL,
  token_id                 VARCHAR(64)     NOT NULL,
  balance_delta            BIGINT          NOT NULL,        -- transparent delta
  shielded_balance_delta   BIGINT          NOT NULL DEFAULT 0,   -- NEW; signed
  timestamp                INT UNSIGNED    NOT NULL,
  voided                   BOOLEAN         NOT NULL DEFAULT FALSE,
  PRIMARY KEY (address, tx_id, token_id)
```

PK unchanged: one row per `(address, tx_id, token_id)` regardless of kind mix. The row's `balance_delta` and `shielded_balance_delta` are written together by `updateAddressTablesWithTx` from the totals it just computed for the vertex; either may be zero. `shielded_balance_delta` is signed so spend reversals can write a negative value.

### `wallet_balance` — modified

```
wallet_balance
  wallet_id                  VARCHAR(64)     NOT NULL,
  token_id                   VARCHAR(64)     NOT NULL,
  -- existing transparent columns
  unlocked_balance           BIGINT UNSIGNED NOT NULL DEFAULT 0,
  locked_balance             BIGINT UNSIGNED NOT NULL DEFAULT 0,
  timelock_expires           INT UNSIGNED    NULL,
  transactions               INT UNSIGNED    NOT NULL DEFAULT 0,
  total_received             BIGINT UNSIGNED NOT NULL DEFAULT 0,
  -- existing authority columns unchanged
  -- new shielded columns
  unlocked_shielded_balance  BIGINT UNSIGNED NOT NULL DEFAULT 0,
  locked_shielded_balance    BIGINT UNSIGNED NOT NULL DEFAULT 0,
  total_shielded_received    BIGINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (wallet_id, token_id)
```

Notes:

- Column-sign rationale mirrors `address_balance`: all balance and total columns are `BIGINT UNSIGNED`, matching the existing transparent convention on master. Spend-reversal helpers use UPDATE-only SQL to dodge MySQL's strict-mode rejection of negative `VALUES()` against UNSIGNED columns. `total_shielded_received` tracks currently-valid lifetime receipts (decremented on void of a recovered shielded output, matching the transparent `total_received` semantic).
- `transactions` is a combined count. A single tx that touches both kinds for the same token increments by 1, not 2 — matching the natural reading of "transactions for this token in this wallet".
- Authority bits stay (transparent only).
- `timelock_expires` stays a single column. The timelock-expiry sweep is unified (one SQL query across `tx_output` regardless of `mode`; see the handlers section below), so a per-kind "next expiry" cache has no consumer — the scheduler reads one row, no per-kind breakdown is exposed by the API, and a single `MIN(timelock)` aggregate over locked rows is the right cache value.

### `wallet_tx_history` — modified

```
wallet_tx_history
  wallet_id              VARCHAR(64)     NOT NULL,
  tx_id                  VARCHAR(64)     NOT NULL,
  token_id               VARCHAR(64)     NOT NULL,
  balance_delta          BIGINT          NOT NULL DEFAULT 0,           -- transparent delta
  shielded_balance_delta BIGINT          NOT NULL DEFAULT 0,           -- NEW
  timestamp              INT UNSIGNED    NOT NULL,
  voided                 BOOLEAN         NOT NULL DEFAULT FALSE,
  PRIMARY KEY (wallet_id, tx_id, token_id),
  INDEX idx_token_timestamp (token_id, timestamp DESC)
```

One row per `(wallet_id, tx_id, token_id)` regardless of kind mix. The handler that emits the history row writes both columns from the totals computed by `updateWalletTablesWithTx`.

### `token` — modified

```
token
  id                     VARCHAR(64)     NOT NULL PRIMARY KEY,
  name                   VARCHAR(30)     NOT NULL,
  symbol                 VARCHAR(5)      NOT NULL,
  -- existing metadata columns unchanged
  total_supply           BIGINT UNSIGNED NOT NULL DEFAULT 0,             -- NEW
```

`SUM(tx_output.value)` is structurally wrong once a token has any shielded UTXOs (shielded values are encrypted on chain and held as NULL on `tx_output` until the wallet-service owns the scan key and recovers them). A denormalised `total_supply` on `token` is updated in-place via four independent supply-change paths, with **two single-writer authorities**:

1. **Token creation** — `handleTokenCreated` is the single writer for the initial supply. It reads `initial_amount` directly from the `TOKEN_CREATED` event payload (`TokenCreatedEventSchema.initial_amount`) and sets `total_supply` to that value via `setTokenTotalSupply`. The wire payload is authoritative — no SUM over `tx_output` is needed, which also removes an implicit ordering dependency on transparent-output materialization.
2. **Block reward** (HTR only) — every accepted block increments `total_supply` for HTR by the block reward.
3. **Mint / melt** — gated on the vertex containing no shielded outputs and at least one authority output. The signed delta is `sum(non-burn outputs[token].value) − sum(inputs[token].value)` — burn-address outputs are **excluded** from the outputs sum so this path reflects the authority-driven supply change only.
4. **Burn sweep** — gated on the vertex containing no shielded outputs. Every transparent output sent to the burn address subtracts its `value` from that token's `total_supply`. Always runs when the gate is satisfied, regardless of whether the tx carried an authority output, since pure burns (no authority) reach `total_supply` only through this path.

Paths 2-4 run inside `applyTokenSupplyUpdates` (called by `handleVertexAccepted`). They are the single writer for supply *deltas* on tokens that already exist. The function never inserts new `token` rows: `incrementTokenTotalSupply` is UPDATE-only, so a delta against a not-yet-created token row no-ops naturally — `handleTokenCreated` is the only inserter, and arrives after `handleVertexAccepted` for the creation tx, then sets the absolute value from the wire.

Paths 3 and 4 are cleanly disjoint: Path 3 is "authority-driven supply change" and Path 4 is "destruction." A mint/melt tx that also burns runs both paths and the deltas compose without double-counting (the burn is excluded from Path 3 and subtracted by Path 4); a pure burn tx runs only Path 4.

The gate on `shielded_outputs.length === 0` is a v1 deferral: shielded mint/melt accounting requires a robust way to identify the authority-driven delta from blinded outputs, which is a follow-up. Until then, any vertex with shielded outputs leaves every involved token's `total_supply` untouched — `getTotalSupply` returns a stale (lower-bound) value for such tokens until the follow-up lands.

Void/unvoid runs paths 2-4 with the sign reversed. Path 1 is reversed by `handleVertexRemoved` when the creation tx is removed during a reorg.

## Event ingestion

Confirmed wire format (fullnode-team alignment, 2026-05-04):

```ts
// packages/daemon/src/types/event.ts (additions)

// `mode` is a numeric discriminator on the wire (1 = AMOUNT_SHIELDED, 2 = FULLY_SHIELDED)
// matching tx_output.mode for shielded rows. Transparent outputs in outputs[] have no `mode`
// field on the wire; the daemon writes `mode = 0` when it inserts the row.
// Use the ShieldedOutputMode enum from @wallet-service/common when comparing in code.
export const ShieldedOutputModeSchema = z.union([
  z.literal(1),
  z.literal(2),
]);

const ShieldedDecodedSchema = z.object({
  address: z.string(),                          // base58 spend address (≤34 chars; mirrors transparent shape)
  timelock: z.number().int().optional(),        // unix seconds; absent ⇒ no timelock (mirrors transparent `decoded.timelock`)
});

const BaseShieldedOutputSchema = z.object({
  mode: ShieldedOutputModeSchema,
  commitment: z.string(),         // hex
  range_proof: z.string(),        // hex
  script: z.string(),             // hex (kept raw for archival; not parsed by the daemon)
  ephemeral_pubkey: z.string(),   // hex
  decoded: ShieldedDecodedSchema,
});

// Note: the shielded wire format does NOT carry a top-level `locked` flag (unlike
// transparent `output.locked`). The daemon derives `locked` locally for shielded
// outputs from `decoded.timelock` and `vertex.heightlock` against the current chain
// time/height — see the `handleVertexAccepted` pseudocode below.

export const AmountShieldedOutputSchema = BaseShieldedOutputSchema.extend({
  mode: z.literal(1),
  token_data: z.number().int(),
});

export const FullyShieldedOutputSchema = BaseShieldedOutputSchema.extend({
  mode: z.literal(2),
  asset_commitment: z.string(),
  surjection_proof: z.string(),
});

export const ShieldedOutputSchema = z.discriminatedUnion('mode', [
  AmountShieldedOutputSchema,
  FullyShieldedOutputSchema,
]);

// Inputs already carry `spent_output` on the wire. With shielded outputs in scope,
// `spent_output` can now be a transparent OR shielded output. We model it as a
// discriminated union: transparent shapes are tagged with a synthetic `mode = 0`
// (or carry no `mode`, depending on what the fullnode emits — both forms parse
// to the same discriminated union via a preprocessor that defaults missing
// `mode` to 0).
const TransparentSpentOutputSchema = ExistingOutputSchema.extend({
  mode: z.literal(0).default(0),
});

export const SpentOutputSchema = z.discriminatedUnion('mode', [
  TransparentSpentOutputSchema,
  AmountShieldedOutputSchema,
  FullyShieldedOutputSchema,
]);

export const InputSchema = ExistingInputSchema.extend({
  spent_output: SpentOutputSchema,
});

export const VertexEventSchema = ExistingVertexEventSchema.extend({
  shielded_outputs: z.array(ShieldedOutputSchema).default([]),
  inputs: z.array(InputSchema),
});
```

**Implication.** The daemon now has the full prior-output data on every input — including `mode`, `decoded.address`, and (for shielded) the crypto bytes. It does **not** need to read `tx_output` to learn the kind of the output being spent; the wire payload is self-describing. The `tx_output` row is still updated (`spent_by` set), but the **dispatch on `mode` for balance reversal reads `input.spent_output.mode` directly** rather than re-querying.

## Daemon `handleVertexAccepted` extension

The ordering is structured around two grains: address-grain involvement (kind-agnostic, no DB lookups) and per-token bookkeeping (kind-aware, only when token+value are known). The shielded credit and reversal helpers that used to live as parallel `applyShielded*` / `reverseShielded*` chains are folded into the unified `updateAddressTablesWithTx` / `updateWalletTablesWithTx` dispatch — the same balance map carries both transparent and shielded contributions per token on the same `(address, token_id)` row.

Pseudocode for the modified handler in `packages/daemon/src/services/index.ts`:

```ts
async function handleVertexAccepted(db, txn, vertex) {
  // 1. Block-specific bookkeeping (miner + heightlock). Existing logic.
  if (isBlock(vertex)) await db.upsertMinerAndHeightlock(txn, vertex);

  // 2. Add tx to transaction table. Existing logic.
  await db.upsertTransaction(txn, vertex);

  // 3-4. Single output loop spanning both kinds. Mark locked and insert all rows
  // (transparent + shielded) into tx_output; shielded rows also write the satellite.
  const concatenated = [
    ...vertex.outputs.map((o, i) => ({ kind: 'transparent', output: o, idx: i })),
    ...vertex.shielded_outputs.map((o, i) => ({
      kind: 'shielded',
      output: o,
      idx: vertex.outputs.length + i,
    })),
  ];
  for (const { kind, output, idx } of concatenated) {
    const timelock = output.decoded.timelock ?? null;
    const locked = (kind === 'transparent') ? output.locked
                                             : isOutputLocked(timelock, vertex.heightlock, vertex);
    await db.insertTxOutput(txn, {
      tx_id: vertex.tx_id, index: idx,
      mode: kind === 'transparent' ? 0 : output.mode,
      address: output.decoded.address,
      value: kind === 'transparent' ? output.value : null,
      token_id: kind === 'transparent' ? tokenUid(output.token_data, vertex)
              : output.mode === 1   ? tokenUid(output.token_data, vertex)
              :                       null,
      authorities: kind === 'transparent' ? (output.token_authorities ?? 0) : 0,
      timelock, heightlock: vertex.heightlock, locked, voided: false,
      recovery_state: kind === 'transparent' ? null : 'unowned',
    });
    if (kind === 'shielded') {
      await db.insertShieldedTxOutputData(txn, vertex.tx_id, idx, output);
      // Upsert the shielded ownership row on the unified `address` table with
      // bip32_account = 1. Does NOT write `shielded_address` / `scan_privkey` /
      // `catchup_state` (those need scan_pubkey + spend_pubkey, populated by the
      // wallet-registration path). Does NOT touch `transactions` — that's the
      // canonical involvement bumper's job at step 6.
      await db.upsertShieldedAddressObservation(txn, output.decoded.address);
    }
  }

  // 5. Compute the involved-addresses set from wire data — no DB lookups.
  // Sources: inputs' spent_output.decoded.address (the address of the UTXO being
  // spent, transparent or shielded), transparent outputs' decoded.address, shielded
  // outputs' decoded.address, and nano-contract header addresses.
  const involved = getInvolvedAddresses(vertex.inputs, vertex.outputs, vertex.shielded_outputs, vertex.headers);

  // 6. Bump `address.transactions` once per involved address. Single canonical
  // writer for the involvement counter (see § Invariants).
  await db.bumpAddressInvolvement(txn, involved);

  // 7. Shielded rewind for owned outputs. After this loop, owned recovered rows
  // carry their value+token in tx_output; the recovery_state moves from 'unowned'
  // to 'recovered' or 'recovery_failed'.
  const recovered = [];  // [{ idx, address, value, token_id, locked }]
  for (const { idx, output } of shieldedOnly(concatenated)) {
    const owned = await db.findShieldedAddressOwnership(txn, output.decoded.address);
    if (!owned?.wallet_id) continue;
    try {
      const r = (output.mode === 1)
        ? rewindAmountShieldedOutput(owned.scan_privkey, output.ephemeral_pubkey,
                                     output.commitment, output.range_proof,
                                     tokenUid(output.token_data, vertex))
        : rewindFullShieldedOutput(owned.scan_privkey, output.ephemeral_pubkey,
                                   output.commitment, output.range_proof,
                                   output.asset_commitment);
      if (output.mode === 2) verifyAssetCommitment(r, output.asset_commitment);
      await db.markTxOutputRecovered(txn, vertex.tx_id, idx, {
        value: r.value,
        ...(r.tokenUid !== undefined ? { token_id: r.tokenUid } : {}),
      });
      recovered.push({
        idx, address: output.decoded.address, value: r.value,
        token_id: r.tokenUid ?? tokenUid(output.token_data, vertex),
        locked: isOutputLocked(output.decoded.timelock ?? null, vertex.heightlock, vertex),
      });
    } catch (e) {
      await db.markTxOutputRecoveryFailed(txn, vertex.tx_id, idx);
      emitAlert('shielded_recovery_failed', { tx_id: vertex.tx_id, index: idx });
    }
  }

  // 8. Mark spent inputs — kind-agnostic, uses tx_id+index only.
  await db.updateTxOutputSpentBy(txn, vertex.inputs, vertex.tx_id);

  // 9. Build the unified per-token balance map. Each entry carries BOTH transparent
  // and shielded contributions for the (address, token) pair. Sources:
  //   - Transparent inputs: prepared inputs.
  //   - Transparent outputs: prepared outputs.
  //   - Shielded inputs (owned, recovered): looked up from local tx_output rows
  //     (the wire payload doesn't carry value or — for FullyShielded — token);
  //     contribute NEGATIVE shielded delta.
  //   - Shielded inputs (unowned or recovery_failed): skipped — the address's
  //     involvement bump fired at step 6.
  //   - Shielded outputs (just-recovered above): contribute POSITIVE shielded delta.
  //   - Shielded outputs (unowned or recovery_failed): skipped.
  // The map only contains (address, token) pairs where token+value are both known.
  const balanceMap = await getUnifiedBalanceMap(
    txn, vertex.inputs, vertex.outputs, recovered, vertex.headers,
  );

  // 10. Single dispatch: writes both transparent and shielded column families on the
  // same (address, token) row, bumps `address_balance.transactions` once per entry.
  await updateAddressTablesWithTx(txn, vertex.tx_id, vertex.timestamp, balanceMap);

  // 11-12. Build wallet map, dispatch. Same shape, per-token only (no wallet-grain
  // involvement counter — `wallet.transactions` is intentionally not a column).
  const addressWalletMap = await db.getAddressWalletInfo(txn, [...balanceMap.keys()]);
  const walletBalanceMap = getWalletBalanceMap(addressWalletMap, balanceMap);
  await updateWalletTablesWithTx(txn, vertex.tx_id, vertex.timestamp, walletBalanceMap);

  // 13. Token-supply deltas for mint/melt/burn (block reward path is separate above).
  // Token CREATION supply is handled by handleTokenCreated using the wire's
  // initial_amount; applyTokenSupplyUpdates here only mutates existing token rows
  // (incrementTokenTotalSupply is UPDATE-only, so a delta against a not-yet-created
  // token row naturally no-ops). The shielded-output-presence gate stays — an
  // unobserved shielded receive's value is invisible from the wire and must not
  // be folded into transparent supply math.
  await applyTokenSupplyUpdates(txn, vertex);

  await enqueueNewTxEvent(vertex);
}
```

`tokenUid` resolves the 1-byte `token_data` to its 32-byte UID by reading the vertex's `tokens[]` array (existing pattern). `isOutputLocked(timelock, heightlock, vertex)` is the locally-computed equivalent of the wire's `output.locked` flag used on the transparent path — it returns true iff the timelock is in the future relative to the daemon's clock OR the vertex's heightlock is above the current chain height. The transparent path doesn't need this helper because the fullnode emits `output.locked` pre-computed on every transparent output; the shielded wire format does not.

`getUnifiedBalanceMap` (a new helper in `packages/daemon/src/utils/wallet.ts`) does two extra DB lookups beyond what `getAddressBalanceMap` did before: one to read the local `tx_output` rows for shielded inputs (so it can find their recovered value+token). Cost is bounded by the number of shielded inputs per vertex (typically 0-1) and runs inside the same per-vertex transaction.

Critical invariant: everything runs inside a single per-vertex DB transaction. The existing `handleVertexAccepted` transaction wrapper is reused unchanged.

## `updateAddressTablesWithTx` and `updateWalletTablesWithTx` — kind-aware

These two functions take the **unified per-token balance map** built by `getUnifiedBalanceMap` and write both transparent and shielded column families on the same `(address, token_id)` row (and the same `(wallet_id, token_id)` row at the wallet grain) in a single statement.

`updateAddressTablesWithTx` is driven by the balance map only — it does NOT bump the address-grain involvement counter. That bump is the canonical writer at step 6 of `handleVertexAccepted` (the `bumpAddressInvolvement` call), driven by the involved-addresses set rather than the balance map; this keeps the involvement bump independent of token knowledge.

Pseudocode:

```ts
async function updateAddressTablesWithTx(txn, txId, timestamp, addressBalanceMap) {
  // For every (address, token_id) touched by this vertex with known token+value:
  for (const [address, perToken] of addressBalanceMap.entries()) {
    for (const [tokenId, balance] of perToken.entries()) {
      // INSERT...ON DUPLICATE KEY UPDATE in one statement, writing both column
      // families on the unified row. The balance object carries both transparent
      // (unlockedAmount, lockedAmount, totalAmountSent) and shielded
      // (unlockedShieldedAmount, lockedShieldedAmount, totalShieldedReceived)
      // contributions. Authority bits stay populated for the transparent columns;
      // shielded outputs cannot carry authority.
      await db.upsertAddressBalanceRow(txn, address, tokenId, balance);
    }
  }

  // Write address_tx_history with both deltas on the same row. balance_delta
  // sums the transparent contributions; shielded_balance_delta sums the shielded
  // contributions. Either may be zero. PK is (address, tx_id, token_id) — one
  // row per pair, regardless of kind mix.
  await db.writeAddressTxHistory(txn, txId, timestamp, addressBalanceMap);
}
```

`updateWalletTablesWithTx` is structurally identical at the wallet grain — writes both column families on the same `(wallet_id, token_id)` row, and both deltas on the same `wallet_tx_history` row. It does NOT touch a wallet-grain involvement counter (no `wallet.transactions` column exists by design).

All shielded balance writes — credit and reversal alike — flow through this single dispatch. There are no separate `applyShielded*` or `reverseShielded*` helpers; the column-family routing is internal to the per-row INSERT…ON DUPLICATE KEY UPDATE statement.

## Spend / consume tracking

A transaction input that consumes a shielded output uses the same `(prev_tx_id, prev_index)` reference as a transparent input, with `prev_index` indexing into the **concatenation** of the previous transaction's `outputs[]` and `shielded_outputs[]`, in that order. Because `tx_output.index` stores the concatenated index for shielded rows just as it does for transparent rows, the dereference is **a single indexed lookup against `tx_output`**:

```ts
async function markTxOutputSpent(txn, input, consumingTxId) {
  const updated = await txn.query(`
    UPDATE tx_output
    SET spent_by = ?
    WHERE tx_id = ? AND index = ?
  `, [consumingTxId, input.prev_tx_id, input.prev_index]);

  if (updated.affectedRows === 0) {
    throw new Error(`input references non-existent output ${input.prev_tx_id}:${input.prev_index}`);
  }
  // The kind of the spent output is already in input.spent_output.mode on the wire
  // (see § Event ingestion), so the balance-reversal step can read mode directly
  // from the event payload without re-querying tx_output.
}
```

**No fallthrough query, no two-table search.** This is the central simplification the unified storage buys: input resolution is symmetric across kinds at the SQL level.

**`spent_output` on the wire.** Every input carries a `spent_output` field containing the full prior output (transparent or shielded), as defined in § *Event ingestion*. This means:

- The daemon can dispatch on `input.spent_output.mode` to decide whether the input is transparent or shielded without an extra `SELECT mode FROM tx_output WHERE …` round-trip.
- If for any reason the local `tx_output` row is missing (e.g., a transient ingestion gap), the wire payload still tells the daemon enough to log the inconsistency precisely, including the kind of the output that was supposed to be there.

**Shielded-input balance contribution requires a local lookup.** The wire `spent_output` for shielded inputs does **not** carry the recovered value (always) or the recovered token (FullyShielded only). So contributing a shielded input to the unified balance map requires reading the local `tx_output` row (one `SELECT` per shielded input):

- If the row has `recovery_state = 'recovered'` and is owned by this wallet, its `value` and `token_id` are populated; contribute a NEGATIVE shielded delta to the `(address, token_id)` entry of the balance map.
- If the row has `recovery_state IN ('unowned', 'recovery_failed')`, skip from the balance map — the address-grain involvement bump at step 6 of `handleVertexAccepted` is the only bookkeeping needed.

This same shape applies symmetrically to the void/reorg path (§ *Reorg, void, unvoid*): the void path also reads the local row to know what shielded balance to reverse, with the same recovery-state filter.

**Cross-tx consumption invariant.** The fullnode is the authoritative validator for "an input cannot reference a shielded output that doesn't exist." The wallet-service treats a `markTxOutputSpent` miss as an integrity error (logged + alerted) rather than a correctness failure on its own. The presence of `input.spent_output` lets the alert payload include the full context of the missing prior output.

## Reorg, void, unvoid

Two distinct void code paths exist in the codebase today, in two different layers and triggered by two different events. The design extends both **in-place**, not duplicated. Per-event dispatch is unchanged: any given event fires one path or the other, never both, so the two are disjoint at runtime.

### Path A — daemon `handleVoidedTx` (per-tx void event)

Lives in `packages/daemon/src/services/index.ts`. Triggered when the fullnode emits a `TX_VOIDED` metadata-diff event for a single tx. Wraps a DB transaction around `voidTx` (`db/index.ts`), which today does:

1. `voidTransaction` — mark the row voided on the `transaction` table.
2. `markUtxosAsVoided` — UPDATE `tx_output` SET `voided = TRUE` for this tx's outputs.
3. Compute `addressBalanceMap` from this tx's inputs and outputs.
4. `voidAddressTransaction` / `voidWalletTransaction` — batched UPDATEs with `op: 'subtract'` reversing the original delta on the relevant balance columns, plus `transactions -= 1` and authority/timelock bookkeeping.
5. `unspendUtxos` — clear `spent_by` on inputs spent by this tx.
6. Token cleanup for any tokens created by this tx.

The shielded extension is **the same unified balance map** that the ingest path uses, run with negated signs. Concretely:

- The void path computes the **unified per-token balance map** for the voided vertex, sourcing shielded contributions from local `tx_output` rows (which still exist with their original `value` / `token_id` / `recovery_state`, just about to be marked voided). For shielded inputs the lookup reads the spent UTXO's row; for shielded outputs the lookup reads the row being voided.
- `voidAddressTransaction` and `voidWalletTransaction` accept the unified map and write both column families in a single UPDATE per row, mirroring `updateAddressTablesWithTx` / `updateWalletTablesWithTx` on the ingest path but with subtractive deltas.
- `total_received` and `total_shielded_received` are decremented here too, mirroring the transparent path's existing behavior. Unrecovered shielded outputs (recovery_state ≠ 'recovered') never contributed to `total_shielded_received` and so don't decrement it on void.
- Address-grain involvement (`address.transactions`) is also decremented here, by one per involved address in the voided vertex — same involved-addresses set the ingest path computed from the wire, just negated.
- `markUtxosAsVoided` is unchanged structurally — it works on `tx_output` rows uniformly, transparent and shielded.

`handleUnvoidedTx` is the mirror — same unified map, forward-signed.

### Path B — wallet-service reorg sweep (`rebuildAddressBalancesFromUtxos`)

Lives in `packages/wallet-service/src/commons.ts` (`handleVoided` at the per-tx wrapper layer; `handleReorg` for chain rewinds) and `packages/wallet-service/src/db/index.ts` (`rebuildAddressBalancesFromUtxos`, `getAffectedAddressTotalReceivedFromTxList`). Triggered by reorg detection or the wallet-service-side single-tx void wrapper. Today it:

1. BFS-walks affected txs (`handleVoidedTxList`), marking each `tx_output.voided = TRUE` and unspending inputs along the way. Accumulates an `affectedUtxoList`.
2. Collects the set of `(addresses, txIds)` touched.
3. Calls `rebuildAddressBalancesFromUtxos`:
   - Zeros out every `address_balance` row for the affected addresses.
   - Recomputes from `tx_output` via two `INSERT ... ON DUPLICATE KEY UPDATE` SUMs (one for unlocked, one for locked).
   - Re-derives `transactions` (`getAffectedAddressTxCountFromTxList`) and `total_received` (`getAffectedAddressTotalReceivedFromTxList`) from `address_tx_history`.
4. `validateAddressBalances` cross-checks the result.

The shielded extension is **kind-aware SQL added in place**:

- The zero-out UPDATE adds the new shielded columns to its `SET` clause (`unlocked_shielded_balance = 0`, `locked_shielded_balance = 0`).
- Two additional INSERT-on-duplicate-update blocks SUM `value` for shielded rows: one for `mode IN (1, 2) AND recovery_state = 'recovered' AND locked = FALSE` (writes `unlocked_shielded_balance`), one for `mode IN (1, 2) AND recovery_state = 'recovered' AND locked = TRUE` (writes `locked_shielded_balance`). Unrecovered shielded rows have `value IS NULL` and naturally drop out of the SUM.
- `getAffectedAddressTotalReceivedFromTxList` today reads `SUM(value) FROM tx_output WHERE tx_id IN (...) AND voided = TRUE GROUP BY address, token_id` to produce the *diff to subtract* from the old `total_received`. The shielded extension adds a parallel SUM in the same query filtered on `mode IN (1, 2) AND recovery_state = 'recovered'` to compute the equivalent diff for `total_shielded_received`. `rebuildAddressBalancesFromUtxos` then subtracts both diffs from the corresponding pre-recompute values it cached at the start of the helper.

### Other handlers

- `handleVertexRemoved`: DELETE `tx_output` rows for the vertex; the FK on `shielded_tx_output_data` cascades. Whatever balance reversal Path A would have done for that vertex's contributions runs once via the same kind-dispatched helpers.
- `handleReorgStarted`: no kind-specific logic; reorgs are realised by the per-vertex remove/void calls that follow.
- `unlockTimelockedUtxos` (periodic timelock-expiry sweep): `getExpiredTimelocksUtxos(now)` returns expired rows from `tx_output` *regardless of `mode`*. `unlockUtxos(utxos)` then:
  1. Batches a single `dbUnlockUtxos(utxos)` to flip `locked = FALSE` on every row.
  2. Partitions the input by `(mode, recovery_state)` and runs two batched balance flows — one for transparent rows, one for recovered shielded rows — through the same kind-aware `updateAddressLockedBalance` / `updateWalletLockedBalance` helpers used by the receive/spend paths. Unowned (or recovery-failed) shielded rows get their row-level `locked = FALSE` flip but no balance write; when the wallet later registers and recovery succeeds, the recovery path reads the row's current `locked = FALSE` and credits `unlocked_shielded_balance` directly. No follow-up unlock is needed.

**Why this is the central argument for the design.** A parallel-tables alternative would add a `…ShieldedOutputs(vertex)` companion call to every one of the four vertex handlers above *plus* a parallel `unlockShieldedTimelockedUtxos` sweep job *plus* a parallel `rebuildShieldedAddressBalancesFromUtxos`, each performing the same arithmetic on parallel tables. Even with helpers that share an implementation, the *call sites* multiply, and every future change to the handlers would have to land in two places at once. With this design, the call sites stay the same; the **dispatch on `mode` lives inside the balance-update functions** (and the kind-aware SUMs live inside the recompute helper), which is exactly the layer that already handles the per-row variance the transparent path requires (authority bits, locked vs unlocked, timelock expiry).

## Push notifications and WebSocket updates

Two channels exist today:

- Push notifications (FCM via SQS) — `packages/daemon/src/services/pushNotification` and the alerts subsystem.
- WebSocket updates to connected wallet clients — `packages/wallet-service/src/ws/`.

This design keeps the **single** `'new-tx'` event. Its payload gains optional shielded fields:

```jsonc
{
  "wallet_id": "…",
  "tx_id":     "…",
  "balance_changes_by_token": [
    {
      "token_id":               "00",
      "balance_delta":          -50,   // transparent
      "shielded_balance_delta": 150    // shielded
    }
  ],
  "shielded_outputs_meta": [           // optional, summary only — no crypto material
    { "index": 5, "mode": 1 }
  ]
}
```

Old clients that don't read the shielded fields see the existing payload shape. Clients that subscribe to `'new-tx'` for a wallet that received shielded outputs see one event per tx, not two.

Push payloads omit any cryptographic material (commitments, ephemeral pubkeys); they carry only the user-visible summary.

## Mempool

Shielded outputs in mempool transactions are handled exactly as transparent mempool outputs are today. No special handling is added. The same `handleVertexAccepted`-equivalent path runs for mempool ingestion, and the same unified loop applies.

## Code organisation

- `packages/daemon/src/services/index.ts` — `handleVertexAccepted`, `handleVoidedTx`, `handleUnvoidedTx`, `handleVertexRemoved` are extended in-place. A new module `packages/daemon/src/services/shielded/recovery.ts` holds the rewind wrapper and ownership-lookup helper; it is **called from** the existing handlers but doesn't replace them.
- `packages/daemon/src/db/index.ts` — `insertTxOutput`, `markTxOutputSpent`, `updateAddressTablesWithTx`, `updateWalletTablesWithTx` gain kind-awareness. New helpers `insertShieldedTxOutputData`, `upsertShieldedAddressObservation` (writes to the unified `address` table with `bip32_account = 1`), `findShieldedAddressOwnership` (SELECT from `address` filtered on `bip32_account = 1`), `applyShieldedAddressBalance`, `reverseShieldedAddressBalanceOnSpend`, `markTxOutputRecovered`, `markTxOutputRecoveryFailed` are added but live next to the existing helpers, not in a separate `shielded.ts` namespace.
- `packages/daemon/src/crypto/ctRewind.ts` — thin wrapper around `@hathor/ct-crypto-node` (centralises error handling and TypeScript types).
- `packages/common/src/types.ts` and `packages/daemon/src/types/event.ts` — extend the shared event/vertex schema types with the new top-level `shielded_outputs[]` field and its mode-discriminated entries.

There is **no** script-parsing module: the fullnode delivers `decoded.address` per shielded output. There is **no** encryption helper: scan-side secrets are stored in plaintext (encryption-at-rest is out of scope for v1).

# Drawbacks
[drawbacks]: #drawbacks

- **`tx_output` becomes a wider, more complex table.** Two new columns (`mode`, `recovery_state`), two columns made nullable (`value`, `token_id`), and several queries need an explicit `WHERE mode = 0` filter to remain transparent-scoped. The migration plan calls out the call sites, but the surface to audit is real.
- **Balance functions branch on `mode`.** `updateAddressTablesWithTx` and `updateWalletTablesWithTx` gain conditional logic. The branches are simple (column dispatch), but they live in code paths that today have no per-row conditionality of this kind.
- **A bad shielded write can corrupt the transparent path.** If a bug writes a shielded delta to `wallet_balance.unlocked_balance` instead of `unlocked_shielded_balance`, the visible transparent balance is wrong even for transparent-only consumers. Physical-isolation designs prevent that class of bug by construction; this design relies on test coverage of the dispatch logic.
- **CPU on the daemon.** Each ownership match triggers a ~1 ms range-proof rewind in-line. The daemon is a long-running process with budget for this, but a sudden spike (e.g., an exchange registering many shielded wallets at once) could lengthen per-vertex processing. Mitigation: the rewind is per-output, not per-vertex, so backpressure is bounded by `outputs-per-tx × matches-per-output`.
- **A second NAPI dependency** (`@hathor/ct-crypto-node`) on the daemon. Mitigation: wrapped behind `crypto/ctRewind.ts` for centralised error handling.
- **Plaintext scan secrets in the DB.** Encryption-at-rest is out of scope. Mitigation: encryption can be added later without changing the schema layout — column types and widths leave room for ciphertext.
- **Schema breadth.** One new table (`shielded_tx_output_data`) plus six modified tables. Modifications are additive; the migration is reversible by dropping the new columns and the satellite table. `address` carries four new nullable-or-defaulted columns whose values are only meaningful for `bip32_account = 1` rows.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why one `tx_output` table instead of a parallel shielded outputs table?

The argument for **two physical tables** is isolation: a bug in shielded handling cannot corrupt rows the transparent code path reads, and shielded ingestion can be added without touching the hot `tx_output` table at all.

The argument for **one table** is handler simplicity: reorg, void, unvoid, vertex-removal, and input-consumption code paths are exactly the same arithmetic and don't need duplication. The point of `tx_output.mode` is to make the discrimination explicit in queries that care, and invisible in queries that don't (which is most of them).

This design takes the second trade-off. Reorg correctness is the most sensitive surface in the daemon; keeping one set of reorg/void/unvoid handlers — with kind dispatch pushed down into the balance-update layer — minimises what has to stay correct in lockstep. The risk that a bad shielded write corrupts transparent rows is real and is mitigated by test coverage of the dispatch logic (see § *Drawbacks*).

## Why a 1:1 satellite for crypto bytes instead of inlining them on `tx_output`?

Inlining would balloon every transparent row with NULL columns that consume disk. Range proofs and surjection proofs are large (~770–930 bytes of cryptographic material per shielded output). A 1:1 satellite keeps the hot `tx_output` table narrow.

The FK with `ON DELETE CASCADE` keeps lifecycle parity automatic: deleting a `tx_output` row on `handleVertexRemoved` removes the satellite row in the same statement.

## Why fold shielded ownership into `address` instead of a separate `shielded_address` table?

An earlier revision used a separate `shielded_address` table mirroring the transparent `address` table's shape. The two-table design was abandoned once it became clear that:

- The shielded `spend_address` is a P2PKH on a separate BIP32 account (`m/44'/280'/1'`); it shares the same on-chain shape as a transparent address and can in principle receive both shielded *and* transparent outputs.
- The parallel-table design forced a duplicate ownership lookup (`findShieldedAddressOwnership`) that did the same work as the transparent lookup on a sibling table, and introduced a class of subtle bugs where shielded and transparent ownership rows could drift out of sync during reorgs.

Folding into one table with a `bip32_account` discriminator collapses those two lookups into one (a SELECT against `address` with `WHERE bip32_account = 1`) and removes the parallel handler. The cost is that four columns on `address` are only meaningful for `bip32_account = 1` rows and stay NULL for transparent rows — accepted in exchange for handler simplicity.

## Why columns instead of a `kind` discriminator on `address_balance` and `address_tx_history`?

The same `(address, token_id)` row can legitimately carry both transparent and shielded balances (the shielded spend_address is a P2PKH, so it can receive transparent outputs as well as shielded ones). A `kind` PK or label would either split that row into two — forcing every consumer to UNION them back — or be redundant alongside the discriminator that already lives on `tx_output.mode`. Mirroring the `wallet_balance` / `wallet_tx_history` shape (one row, separate `*_balance` and `*_shielded_balance` columns; one row, `balance_delta` and `shielded_balance_delta`) keeps the address grain consistent with the wallet grain.

## Why `wallet_balance` separate columns instead of `wallet_balance.kind` rows?

`wallet_balance` is per-`(wallet_id, token_id)`. A wallet can hold both transparent and shielded balance for the same token simultaneously. If `kind` were part of the PK, a single per-token wallet aggregate would become two rows that consumers must sum. Separate columns let consumers read one row and pick the field they care about (or `?include=split` to get both).

## Why a single `wallet_tx_history` row per `(wallet_id, tx_id, token_id)` instead of two rows by `kind`?

A user querying their history wants one entry per tx, not two. Splitting by `kind` in the PK would surface implementation detail (a mixed tx becomes two rows that the API has to remerge for display) without changing storage size meaningfully.

The two-column layout (`balance_delta`, `shielded_balance_delta`) on a shared row lets the API render `output_kind` per row as `transparent | shielded | mixed` by reading which columns are non-zero.

## Why default the balance API to merged instead of split?

Backward compatibility. Old clients read `balance.unlocked` as a number and the merged total is the natural meaning of "the user's spendable balance". A privacy-aware UI that wants the breakdown opts in via `?include=split`.

The alternative — surfacing the split by default — would break the response shape for old clients and force every consumer to migrate.

## Why store `recovered_value` / recovered `token_id` on `tx_output` instead of on the satellite?

Recovery state is naturally co-located with `mode`. Consumers that want "every recovered output for this wallet" filter `tx_output` by `mode IN (1,2) AND recovery_state = 'recovered'` and find both the discriminator and the result without joining. The satellite is pure cryptographic-bytes storage.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The wire-format and library-stability questions raised during early design were resolved with the fullnode team and ops. The remaining items are local design choices:

1. **Behaviour on `recovery_failed`.** Today's design alerts and continues. An alternative is to halt the daemon (treat as a consensus error). The first behaviour is preferred because a malformed proof would have been rejected by the fullnode upstream; a recovery failure here implies a key/derivation bug, not a data-corruption bug, and halting would be over-broad.

2. **Indexes on `tx_output` after the migration.** With `mode` added, the question is whether the existing single-column `idx_address` index is enough or whether a composite `(mode, address)` index pays off for the catch-up scan (`WHERE mode IN (1,2) AND address IN (...)`). Defer to benchmark on production-shaped data; the design is correct either way.

3. **Whether `tx_output.value` should be SIGNED or UNSIGNED after the change.** Today's column is `BIGINT UNSIGNED`. Making it nullable doesn't change the sign. Confirm with ops that a nullable unsigned bigint is portable across the team's read-replica tooling.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Background batch rewind for catch-up.** When an existing wallet upgrades (the catch-up flow described in the wallet-registration design document), the catch-up scan is naturally async and parallel; this design's synchronous-on-the-hot-path choice does not preclude an out-of-band batch worker, it merely declines to use one for the main pipeline.
- **Indexing for view-key delegation.** If the upstream RFC's "view key delegation" feature lands, the indexer is naturally positioned to act as the delegated viewer. The schema accommodates this (the per-row `scan_privkey` is already the only secret needed) without further migrations.
- **Nullifier-based consumption tracking.** If phase C of the upstream RFC lands nullifiers, `tx_output` (or the satellite) gains an indexed `nullifier` column and the input-parser changes; the rest of the design is unaffected.
- **Range-proof verification by the indexer.** Currently delegated to the fullnode. If we ever want defense-in-depth, the rewind path can call `verify_range_proof` after rewind for a small extra cost.
- **Shielded mint/melt support.** The in-flight upstream [shielded mint/melt RFC](https://github.com/HathorNetwork/rfcs/blob/feat/shielded-outputs/text/0000-shielded-outputs-mint-melt.md) adds shielded *value* outputs to mint/melt transactions while keeping authority outputs transparent (parent-RFC Rule 7 stays). It repeals Rule 8 (the prohibition on mixing shielded outputs with mint/melt) and introduces two new vertex headers — `MintHeader` (`0x14`) and `MeltHeader` (`0x15`), each carrying a list of `(token_index, amount)` entries that publicly declare per-token supply deltas while keeping recipient sets and per-recipient amounts private. Wallet-service impact when this lands:
  - Daemon parser extended to recognise the new `MintHeader` (`0x14`) and `MeltHeader` (`0x15`).
  - Per-token supply tracking updated to apply the public mint/melt scalars from these headers.
  - Existing shielded recovery flow handles incoming shielded value outputs from mint/melt transactions with no change.
  - Token-creation transactions become eligible to carry shielded outputs (under refined Rule M2); the existing token-creation indexing path is extended to call into the shielded recovery flow when a `shielded_outputs[]` array is present on a TCT.
  - No schema change required for `tx_output` itself; the new work is parsing two new headers and updating supply counters.
