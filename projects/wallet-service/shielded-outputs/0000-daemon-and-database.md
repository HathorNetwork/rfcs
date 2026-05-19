- Feature Name: `wallet_service_shielded_outputs_daemon_schema`
- Start Date: 2026-05-12
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Add shielded-output support to the wallet-service by **extending the existing `tx_output`, `address_balance`, `wallet_balance`, `address_tx_history`, and `wallet_tx_history` tables**. A new `mode` column on `tx_output` discriminates `0 = transparent`, `1 = AMOUNT_SHIELDED`, `2 = FULLY_SHIELDED`. The heavy shielded-only on-chain bytes (commitment, range proof, ephemeral pubkey, script, plus mode-conditional fields) live in a 1:1 satellite table `shielded_tx_output_data` keyed by the same `(tx_id, output_index)` PK as `tx_output`. Shielded ownership lives in a new `shielded_address` table that, like the existing transparent `address` table, has rows for every observed spend_address — ownership fields populate only when a wallet claims the address.

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
       upsert shielded_address row (transactions++; create if new)
     if output is owned:
       update address_balance, wallet_balance, *_tx_history
       (column dispatch on mode)
```

Everything runs inside the existing per-vertex DB transaction.

The contributor must hold four concepts:

1. **`tx_output.mode`** — the single discriminator. `0 = transparent`, `1 = AMOUNT_SHIELDED`, `2 = FULLY_SHIELDED`. Every kind-sensitive query filters on this column. Existing queries that omit it are implicitly transparent-only **only if explicitly filtered**, which is why a migration step adds `WHERE mode = 0` to every existing `tx_output` query that should remain transparent-scoped.

2. **`recovery_state`** — `tx_output.recovery_state` (NULL for `mode = 0`; `unowned` / `recovered` / `recovery_failed` for `mode IN (1, 2)`). For unowned shielded outputs, `value` is always NULL; for `mode = 1` the `token_id` is populated at observe time from `token_data` (visible on the wire), so only `value` is filled by recovery; for `mode = 2`, both `value` and `token_id` are NULL until recovery.

3. **Eager rewind on match.** When a shielded output's `address` matches a `shielded_address` row with non-NULL `wallet_id`, the daemon synchronously calls `rewindAmountShieldedOutput` or `rewindFullShieldedOutput` (from `@hathor/ct-crypto-node`) using the cached per-index `scan_privkey` for that row. On success, the recovered `{ value, token_id }` is written back to the **same** `tx_output` row (`recovery_state = 'recovered'`). Blinding factors are not stored.

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
5. It UPSERTs `shielded_address`: row for Alice's `decoded.address` already exists with `wallet_id = wallet_alice`, `shielded_index = 7`, `scan_privkey = …` because Alice registered shielded keys earlier. The upsert bumps `transactions` from 0 to 1.
6. It sees `wallet_id IS NOT NULL` on the looked-up row → owned. It calls `rewindAmountShieldedOutput(scan_privkey, ephemeral_pubkey, commitment, range_proof, token_uid_from_token_data)`. Returns `{ value: 150 }`.
7. It UPDATEs the `tx_output` row: `value = 150`, `recovery_state = 'recovered'`. (`token_id` is already populated.)
8. It calls `updateAddressTablesWithTx`, which dispatches on `mode = 1`:
   - Bump `address_balance.unlocked_balance += 150` for `(Alice_spend_address, '00')` with `kind = 'shielded'`.
   - Bump `wallet_balance.unlocked_shielded_balance += 150` for `(wallet_alice, '00')`.
   - Append `wallet_tx_history` row with `shielded_balance_delta = 150` (and `balance_delta = 0`).
9. It enqueues a `'new-tx'` event carrying both transparent (empty here) and shielded deltas.
10. The DB transaction commits.

If the same vertex also had transparent outputs, those would be processed in the same loop and contribute to the **same** `wallet_tx_history` row (one row per `(wallet_id, tx_id, token_id)`), with `balance_delta` carrying the transparent delta and `shielded_balance_delta` carrying the shielded delta.

## Worked example: mixed receive (transparent + shielded in one tx)

Bob sends Alice 1.0 HTR via a transparent output **and** 1.5 HTR via an `AMOUNT_SHIELDED` output in the same transaction. Alice's wallet has shielded keys registered.

1. Fullnode emits `NEW_VERTEX_ACCEPTED`. The vertex has `outputs[]` (one entry, 1.0 HTR transparent to Alice) and `shielded_outputs[]` (one entry, 1.5 HTR shielded to Alice).
2. `handleVertexAccepted` builds the concatenated list and iterates:
   - **`outputs[0]` (transparent, concatenated index 0).** INSERTs `tx_output` row with `mode = 0`, `address = Alice_transparent_address`, `value = 100`, `token_id = '00'`, `recovery_state = NULL`. The existing transparent ownership lookup against `address` finds `wallet_id = wallet_alice`. Marked for balance update with `kind = 'transparent'`.
   - **`shielded_outputs[0]` (mode=1, concatenated index 1).** INSERTs `tx_output` row with `mode = 1`, `address = Alice_spend_address`, `value = NULL`, `token_id = '00'` (from `token_data`), `recovery_state = 'unowned'`. INSERTs satellite row with crypto bytes. UPSERTs `shielded_address` (already owned by `wallet_alice` at `shielded_index = 7`). Rewinds → `value = 150`. UPDATEs the `tx_output` row to `value = 150`, `recovery_state = 'recovered'`.
3. `updateAddressTablesWithTx` runs over both rows in the same call, dispatching on `mode`:
   - `address_balance` for `(Alice_transparent_address, '00')`: bump `unlocked_balance += 100`, `kind = 'transparent'`.
   - `address_balance` for `(Alice_spend_address, '00')`: bump `unlocked_balance += 150`, `kind = 'shielded'`.
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
   - **`shielded_outputs[0]` (Bob's, concatenated index 0).** INSERTs `tx_output` row with `mode = 1`, `address = Bob_spend_address`, `value = NULL`, `token_id = '00'`, `recovery_state = 'unowned'`. INSERTs satellite. UPSERTs `shielded_address` for `Bob_spend_address` — row didn't exist before (this is a brand-new observation), so an INSERT runs with `wallet_id = NULL`, `transactions = 1`. Ownership lookup: `wallet_id IS NULL` → unowned, no rewind, no balance update.
   - **`shielded_outputs[1]` (Alice's change, concatenated index 1).** INSERTs `tx_output` with same shape as the prior example. UPSERTs `shielded_address` (Alice's change address is owned at `shielded_index = 12`, say). Rewinds → `value = 250`. UPDATEs `tx_output` to `value = 250`, `recovery_state = 'recovered'`.
2. Then the input loop:
   - **`inputs[0]`.** `markTxOutputSpent(tx_id=T0, index=5)` runs `UPDATE tx_output SET spent_by = <current_tx_id> WHERE tx_id = T0 AND index = 5`. The kind of the consumed output is read from `input.spent_output.mode = 1` directly — no second query.
3. `updateAddressTablesWithTx` and `updateWalletTablesWithTx` aggregate every row touched by the vertex:
   - From the spent input: `-400` shielded delta (Alice's old 4.0 HTR UTXO is now consumed).
   - From the new owned output `shielded_outputs[1]`: `+250` shielded delta (Alice's change).
   - From the new unowned output `shielded_outputs[0]`: no balance impact (unowned).
4. Resulting writes:
   - `address_balance` for `(Alice_spend_address_of_T0_index_5, '00')`: `unlocked_balance -= 400`, `kind = 'shielded'`.
   - `address_balance` for `(Alice_change_spend_address, '00')`: `unlocked_balance += 250`, `kind = 'shielded'`.
   - `wallet_balance` for `(wallet_alice, '00')`: `unlocked_shielded_balance += (-400 + 250) = -150`.
   - `wallet_tx_history`: one row with `balance_delta = 0`, `shielded_balance_delta = -150`.
5. Bob's wallet-service (if he uses one) sees the same vertex independently and runs the same flow against his own `shielded_address` table; for him, Bob's output is owned and recovered, and Alice's change is unowned.

The point of this example: **the input's `spent_output.mode` is the only signal needed to dispatch the balance reversal**, and the `UPDATE tx_output SET spent_by = …` works on shielded rows with no special case.

## Worked example: third-party shielded send (nobody on this wallet-service is involved)

Charlie sends Dave a shielded output. Neither Charlie nor Dave has a wallet registered on this wallet-service. The daemon still observes and persists the data — this is how catch-up works when Dave (or Charlie) later registers.

1. `handleVertexAccepted` iterates `shielded_outputs[]`:
   - INSERTs `tx_output` with `mode = 1`, `address = Dave_spend_address`, `value = NULL`, `token_id = '00'`, `recovery_state = 'unowned'`.
   - INSERTs satellite row with crypto bytes.
   - UPSERTs `shielded_address` for `Dave_spend_address` (new row, `wallet_id = NULL`, `transactions = 1`).
   - Ownership lookup: `wallet_id IS NULL` → no rewind, no balance update.
2. The input loop processes Charlie's spent input the same way: `UPDATE tx_output SET spent_by = …` (Charlie's prior UTXO was observed by the daemon the same way, with NULL ownership).
3. `updateAddressTablesWithTx` runs and finds nothing to do (no `wallet_id` owns any of the touched addresses on this service). No `address_balance` / `wallet_balance` writes.
4. No `'new-tx'` event is emitted to any wallet (no subscriber).
5. The DB transaction commits.

A week later, Dave registers shielded keys. His registration UPSERTs `shielded_address` rows for his first 20 indexes; one of those addresses **is** the row already created at step 1 above, so the UPSERT fills in `wallet_id`, `scan_privkey`, etc., transitioning the row from "observed, unowned" to "owned, catchup pending". The catch-up job then scans `tx_output WHERE mode IN (1,2) AND recovery_state = 'unowned' AND address IN (Dave's claimed set)`, finds the row written at step 1, runs the rewind, and credits Dave's balance retroactively.

## Worked example: FULLY_SHIELDED receive

Alice receives 0.75 of token `T1` via a `FULLY_SHIELDED` output (mode = 2). The token type is also hidden on the wire.

1. `handleVertexAccepted` iterates `shielded_outputs[]`:
   - INSERTs `tx_output` with `mode = 2`, `address = Alice_spend_address`, `value = NULL`, `token_id = NULL` (token is hidden — no `token_data` to read), `recovery_state = 'unowned'`.
   - INSERTs satellite row including `asset_commitment` and `surjection_proof` (the mode-2-only fields).
   - UPSERTs `shielded_address` (Alice's address, owned).
   - Ownership lookup hits. Calls `rewindFullShieldedOutput(scan_privkey, ephemeral_pubkey, commitment, range_proof, asset_commitment)`. Returns `{ value: 75, tokenUid: 'T1', assetBlindingFactor: … }`.
   - Verifies the asset commitment against the recovered token (`verifyAssetCommitment`) — a mandatory step for mode 2.
   - UPDATEs `tx_output`: `value = 75`, `token_id = 'T1'`, `recovery_state = 'recovered'`.
2. `updateAddressTablesWithTx` updates `address_balance` for `(Alice_spend_address, 'T1')`, `kind = 'shielded'`, `unlocked_balance += 75`.
3. `wallet_balance` for `(wallet_alice, 'T1')`: `unlocked_shielded_balance += 75`. If this is the first time wallet_alice has seen token T1, the row is created by the upsert.
4. `wallet_tx_history` row written with `shielded_balance_delta = 75` for `(wallet_alice, tx_id, 'T1')`.

The point of this example: `token_id` lives in `tx_output` and is filled in by recovery for mode 2, whereas for mode 1 it would have been filled at observe time. The handler dispatches on `mode` to know which.

## Worked example: void of a recovered shielded receive

Two days after the first worked example (Alice received 1.5 HTR shielded), the transaction is reorged out and gets voided.

1. `handleVoidedTx` runs for the vertex.
2. UPDATEs every `tx_output` row of the vertex: `voided = TRUE` (affects both transparent and shielded rows uniformly).
3. UPDATEs `wallet_tx_history` and `address_tx_history` rows for the same `tx_id`: `voided = TRUE`.
4. Reverses the balance deltas by re-running `updateAddressTablesWithTx` / `updateWalletTablesWithTx` with a reversing sign. For each `tx_output` row of the vertex:
   - The row carries `mode = 1` and `recovery_state = 'recovered'` (it was recovered on receive).
   - Dispatch on `mode` picks the shielded balance columns. `wallet_balance.unlocked_shielded_balance -= 150` for `(wallet_alice, '00')`. `address_balance.unlocked_balance -= 150` for `(Alice_spend_address, '00')` with `kind = 'shielded'`.
5. Push notification or WebSocket event emitted to wallet_alice indicating the void.

Crucially: this is the **same** `handleVoidedTx` that runs for a transparent void; the only kind-specific work is the column dispatch inside `updateAddressTablesWithTx`. There is no `voidShieldedOutputs` companion handler. The same observation applies to `handleUnvoidedTx` (re-applies) and `handleVertexRemoved` (delete, with FK cascading the satellite row).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Schema changes

### `tx_output` — modified

```
tx_output
  -- existing columns
  tx_id                   VARCHAR(64)     NOT NULL,
  index                   SMALLINT        NOT NULL,  -- concatenated index (outputs ++ shielded_outputs)
  address                 VARCHAR(34)     NOT NULL,  -- decoded.address for both transparent and shielded
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

### `shielded_tx_output_data` — new, 1:1 satellite

Holds the heavy on-chain cryptographic bytes for every shielded `tx_output` row. Keyed by the same composite PK.

```
shielded_tx_output_data
  tx_id              VARCHAR(64)     NOT NULL,
  output_index       SMALLINT        NOT NULL,
  commitment         VARBINARY(33)   NOT NULL,
  range_proof        VARBINARY(1024) NOT NULL,
  script             VARBINARY(1024) NOT NULL,  -- raw, not parsed
  ephemeral_pubkey   VARBINARY(33)   NOT NULL,
  token_data         TINYINT         NULL,    -- non-null iff mode = 1
  asset_commitment   VARBINARY(33)   NULL,    -- non-null iff mode = 2
  surjection_proof   VARBINARY(4096) NULL,    -- non-null iff mode = 2
  PRIMARY KEY (tx_id, output_index),
  FOREIGN KEY (tx_id, output_index) REFERENCES tx_output(tx_id, index) ON DELETE CASCADE
```

Notes:

- The FK with `ON DELETE CASCADE` guarantees `handleVertexRemoved`'s `DELETE FROM tx_output WHERE tx_id = ?` automatically cleans up the satellite — no application-layer second delete.
- `script` is stored raw for archival. The daemon does not parse it.
- `recovery_state` is **not** here — it lives on `tx_output` so consumers can filter by it without joining the satellite. This is the answer to "where do I look to find every owned-but-unrecovered output for catch-up": `SELECT … FROM tx_output WHERE mode IN (1,2) AND recovery_state = 'unowned' AND address IN (…)`.

### `shielded_address` — new

Holds every spend_address observed on chain in a shielded output, regardless of ownership. Ownership fields are NULL until a wallet claims the address.

```
shielded_address
  address           VARCHAR(34)    NOT NULL,  -- on-chain spend address (base58); PK
  wallet_id         VARCHAR(64)    NULL,      -- NULL until owned
  shielded_index    INT UNSIGNED   NULL,      -- NULL until owned
  shielded_address  VARCHAR(100)   NULL,      -- 71-byte payload, base58, ≤100 chars; NULL until owned
  scan_privkey      VARBINARY(32)  NULL,      -- plaintext; NULL until owned
  catchup_state     ENUM('pending','running','done') NULL,  -- NULL until owned
  transactions      INT UNSIGNED   NOT NULL DEFAULT 0,
  created_at        TIMESTAMP      NOT NULL,
  PRIMARY KEY (address),
  UNIQUE KEY uk_wallet_shielded_index (wallet_id, shielded_index),
  INDEX idx_shielded_address (shielded_address),
  INDEX idx_wallet_catchup (wallet_id, catchup_state)
```

Notes:

- This mirrors how the existing transparent `address` table works: rows exist for every observed address; `wallet_id` populates only when a wallet that derives the address registers.
- `transactions` increments on every observed shielded output regardless of ownership. It is the per-address activity counter used for gap-limit decisions during catch-up.
- `(wallet_id, shielded_index)` is UNIQUE. MySQL treats multiple NULLs as distinct, so unowned rows (NULL `wallet_id`) coexist freely.
- `scan_pubkey` and `spend_pubkey` are **not** stored — they are intermediate values during derivation: `scan_pubkey` is recoverable via point multiplication from `scan_privkey`, and `spend_pubkey` is recoverable via BIP32 derivation from `wallet.spend_xpub` at the row's `shielded_index`.

### `address_balance` — modified

```
address_balance
  -- existing columns
  address                 VARCHAR(34)     NOT NULL,
  token_id                VARCHAR(64)     NOT NULL,
  unlocked_balance        BIGINT          NOT NULL DEFAULT 0,
  locked_balance          BIGINT UNSIGNED NOT NULL DEFAULT 0,
  timelock_expires        INT UNSIGNED    NULL,
  transactions            INT UNSIGNED    NOT NULL DEFAULT 0,
  total_received          BIGINT UNSIGNED NOT NULL DEFAULT 0,
  -- existing authority columns unchanged
  -- new column
  kind                    ENUM('transparent','shielded') NOT NULL DEFAULT 'transparent',
  PRIMARY KEY (address, token_id),
  INDEX idx_kind_token (kind, token_id)
```

Notes:

- `kind` is functionally determined by which ownership table holds the address (`address` vs `shielded_address`) — the two address spaces are disjoint. The column is a denormalised label so consumers can filter without JOINing.
- Authority bits stay populated for transparent rows; always 0 for shielded rows (shielded outputs cannot carry authority).
- `idx_kind_token` accelerates per-kind aggregations and reorg-time scans.

### `address_tx_history` — modified

```
address_tx_history
  address           VARCHAR(34)     NOT NULL,
  tx_id             VARCHAR(64)     NOT NULL,
  token_id          VARCHAR(64)     NOT NULL,
  balance_delta     BIGINT          NOT NULL,
  timestamp         INT UNSIGNED    NOT NULL,
  voided            BOOLEAN         NOT NULL DEFAULT FALSE,
  -- new column
  kind              ENUM('transparent','shielded') NOT NULL DEFAULT 'transparent',
  PRIMARY KEY (address, tx_id, token_id),
  INDEX idx_kind_token_timestamp (kind, token_id, timestamp DESC)
```

PK unchanged: an address is one kind, and a `(address, tx_id, token_id)` triple is a unique movement. `kind` is a label.

### `wallet_balance` — modified

```
wallet_balance
  wallet_id                  VARCHAR(64)     NOT NULL,
  token_id                   VARCHAR(64)     NOT NULL,
  -- existing transparent columns
  unlocked_balance           BIGINT          NOT NULL DEFAULT 0,
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

`SUM(tx_output.value)` is structurally wrong once a token has any shielded UTXOs (shielded values are encrypted on chain and held as NULL on `tx_output` until the wallet-service owns the scan key and recovers them). A denormalised `total_supply` on `token` is updated in-place via four independent supply-change paths:

1. **Token creation** — `total_supply` is set to the initial mint amount when the token-creation tx is processed.
2. **Block reward** (HTR only) — every accepted block increments `total_supply` for HTR by the block reward.
3. **Mint / melt** — gated on the vertex containing no shielded outputs and at least one authority output. The signed delta is `sum(outputs[token].value) − sum(inputs[token].value)`, **including** burn-address outputs in the outputs sum so the authority-driven delta reflects the real supply change.
4. **Burn sweep** — gated on the vertex containing no shielded outputs. Every transparent output sent to the burn address subtracts its `value` from that token's `total_supply`. The mint/melt math includes burns in its outputs sum, so the burn sweep handles destruction as a separate, additive correction (no double-count).

The gate on `shielded_outputs.length === 0` is a v1 deferral: shielded mint/melt accounting requires a robust way to identify the authority-driven delta from blinded outputs, which is a follow-up. Until then, any vertex with shielded outputs leaves every involved token's `total_supply` untouched — `getTotalSupply` returns a stale (lower-bound) value for such tokens until the follow-up lands.

Void/unvoid runs paths 2–4 with the sign reversed. Path 1 is reversed by `handleVertexRemoved` when the creation tx is removed during a reorg.

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
  address: z.string(),  // base58 spend address (≤34 chars; mirrors transparent shape)
});

const BaseShieldedOutputSchema = z.object({
  mode: ShieldedOutputModeSchema,
  commitment: z.string(),         // hex
  range_proof: z.string(),        // hex
  script: z.string(),             // hex (kept raw for archival; not parsed by the daemon)
  ephemeral_pubkey: z.string(),   // hex
  decoded: ShieldedDecodedSchema,
});

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

Pseudocode for the modified handler in `packages/daemon/src/services/index.ts`:

```ts
async function handleVertexAccepted(db, txn, vertex) {
  await db.upsertTransaction(txn, vertex);                              // existing

  // Single output loop spanning both kinds.
  const concatenated = [
    ...vertex.outputs.map((o, i) => ({ kind: 'transparent', output: o, idx: i })),
    ...vertex.shielded_outputs.map((o, i) => ({
      kind: 'shielded',
      output: o,
      idx: vertex.outputs.length + i,
    })),
  ];

  for (const { kind, output, idx } of concatenated) {
    if (kind === 'transparent') {
      await db.insertTxOutput(txn, {
        tx_id: vertex.tx_id, index: idx, mode: 0,
        address: output.decoded.address,
        value: output.value, token_id: tokenUid(output.token_data, vertex),
        authorities: output.token_authorities ?? 0,
        timelock: output.decoded.timelock, heightlock: vertex.heightlock,
        locked: output.locked, voided: false,
        recovery_state: null,
      });
    } else {
      await db.insertTxOutput(txn, {
        tx_id: vertex.tx_id, index: idx, mode: output.mode,
        address: output.decoded.address,
        value: null,
        token_id: output.mode === 1 ? tokenUid(output.token_data, vertex) : null,
        authorities: 0,
        timelock: null, heightlock: vertex.heightlock,
        locked: false, voided: false,
        recovery_state: 'unowned',
      });

      await db.insertShieldedTxOutputData(txn, vertex.tx_id, idx, output);

      // Upsert the shielded_address row for activity tracking.
      // Creates a row with NULL ownership fields if this is the first sighting of the address.
      await db.upsertShieldedAddressObservation(txn, output.decoded.address);

      // Ownership lookup. Single SELECT.
      const owned = await db.findShieldedAddressOwnership(txn, output.decoded.address);
      if (owned?.wallet_id) {
        try {
          const recovered = (output.mode === 1)
            ? rewindAmountShieldedOutput(
                owned.scan_privkey, output.ephemeral_pubkey, output.commitment,
                output.range_proof, tokenUid(output.token_data, vertex),
              )
            : rewindFullShieldedOutput(
                owned.scan_privkey, output.ephemeral_pubkey, output.commitment,
                output.range_proof, output.asset_commitment,
              );
          if (output.mode === 2) verifyAssetCommitment(recovered, output.asset_commitment);

          await db.markTxOutputRecovered(txn, vertex.tx_id, idx, {
            value: recovered.value,
            token_id: recovered.tokenUid ?? output.token_data ? tokenUid(output.token_data, vertex) : null,
          });
        } catch (e) {
          await db.markTxOutputRecoveryFailed(txn, vertex.tx_id, idx);
          emitAlert('shielded_recovery_failed', { tx_id: vertex.tx_id, output_index: idx });
        }
      }
    }
  }

  // Inputs: a single lookup against tx_output regardless of kind. Each input also carries
  // its spent_output on the wire (transparent or shielded), so the kind of the consumed
  // output is known without re-querying.
  for (const input of vertex.inputs) {
    await db.markTxOutputSpent(txn, input, vertex.tx_id);
  }

  // Balance / history updates dispatch on mode — sourced from tx_output for outputs we
  // just wrote, and from input.spent_output.mode for outputs being consumed.
  await updateAddressTablesWithTx(txn, vertex);
  await updateWalletTablesWithTx(txn, vertex);

  await enqueueNewTxEvent(vertex);
}
```

`tokenUid` resolves the 1-byte `token_data` to its 32-byte UID by reading the vertex's `tokens[]` array (existing pattern).

Critical invariant: everything runs inside a single per-vertex DB transaction. The existing `handleVertexAccepted` transaction wrapper is reused unchanged.

## `updateAddressTablesWithTx` and `updateWalletTablesWithTx` — kind-aware

These two functions already do the most intricate balance-arithmetic work in the codebase. The shielded extension adds a switch on `tx_output.mode` (and on `recovery_state` for shielded rows) when computing deltas and writing them back.

Pseudocode:

```ts
async function updateAddressTablesWithTx(txn, vertex) {
  // 1. Fetch every tx_output row affected by this vertex (rows we just wrote, plus
  //    rows newly marked spent_by). Each row carries its `mode` and `recovery_state`.
  const affected = await db.getAffectedTxOutputs(txn, vertex.tx_id);

  // 2. Group by (address, token_id, kind). For unrecovered shielded rows, skip —
  //    we can't update balance without a known value.
  for (const row of affected) {
    if (row.mode === 0) {
      // existing transparent logic, unchanged
      await applyAddressBalanceDelta(txn, row.address, row.token_id, 'transparent',
        signedAmount(row), row.locked);
    } else if (row.recovery_state === 'recovered') {
      await applyAddressBalanceDelta(txn, row.address, row.token_id, 'shielded',
        signedAmount(row), row.locked);
    }
    // mode IN (1,2) && recovery_state IN ('unowned','recovery_failed') => no balance impact.
  }
}
```

`applyAddressBalanceDelta` writes to `address_balance` with the appropriate `kind` label and uses one pair of `unlocked_balance` / `locked_balance` columns (the row's `kind` already says which type of balance it carries).

`updateWalletTablesWithTx` is similar but writes to `wallet_balance` and `wallet_tx_history`. For `wallet_balance` it dispatches on `mode`:

- `mode = 0` → `unlocked_balance` / `locked_balance` columns.
- `mode IN (1, 2)` recovered → `unlocked_shielded_balance` / `locked_shielded_balance` columns.

For `wallet_tx_history`, the per-tx-per-token aggregation produces both `balance_delta` (sum of transparent contributions) and `shielded_balance_delta` (sum of shielded contributions) for the same row.

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

- The daemon can dispatch balance-reversal logic on `input.spent_output.mode` directly, without an extra `SELECT mode FROM tx_output WHERE …` round-trip.
- If for any reason the local `tx_output` row is missing (e.g., a transient ingestion gap), the wire payload still tells the daemon enough to log the inconsistency precisely, including the kind of the output that was supposed to be there.
- Reorg/void paths that need to undo a balance delta read `mode` from the event payload, so they don't have to re-query `tx_output` (which may already be in a transient state during the same transaction).

**Cross-tx consumption invariant.** The fullnode is the authoritative validator for "an input cannot reference a shielded output that doesn't exist." The wallet-service treats a `markTxOutputSpent` miss as an integrity error (logged + alerted) rather than a correctness failure on its own. The presence of `input.spent_output` lets the alert payload include the full context of the missing prior output.

## Reorg, void, unvoid

Each existing handler in `packages/daemon/src/services/index.ts` is extended in-place, not duplicated:

- `handleVoidedTx`:
  1. UPDATE `tx_output` SET `voided = TRUE` WHERE `tx_id = ?` — affects both transparent and shielded rows uniformly.
  2. UPDATE `wallet_tx_history` and `address_tx_history` SET `voided = TRUE` for the same `tx_id`.
  3. Reverse the balance deltas by re-running `updateAddressTablesWithTx` / `updateWalletTablesWithTx` with a reversing sign. Inside those functions, `mode` dispatch picks the right columns.
  4. On the wallet-service side, `rebuildAddressBalancesFromUtxos` recomputes `address_balance` rows for affected addresses. The recompute helper queries (including `getAffectedAddressTotalReceivedFromTxList`) are kind-agnostic at the SQL level: `SUM(value)` skips NULL for unrecovered shielded rows and contributes the recovered portion for recovered ones, and the per-`(address, token)` UPDATE hits the correct kind-discriminated row in `address_balance` (transparent and shielded address spaces are disjoint, so each row is unambiguously one kind).

- `handleUnvoidedTx`: mirror of `handleVoidedTx` (sets `voided = FALSE`; re-applies deltas).

- `handleVertexRemoved`: DELETE `tx_output` rows for the vertex; the FK on `shielded_tx_output_data` cascades. Balance reversal runs once.

- `handleReorgStarted`: no kind-specific logic; reorgs are realised by the per-vertex remove/void calls that follow.

- `unlockTimelockedUtxos` (the periodic timelock-expiry sweep): `getExpiredTimelocksUtxos(now)` returns expired rows from `tx_output` *regardless of `mode`*. `unlockUtxos(utxos)` then:
  1. Batches a single `dbUnlockUtxos(utxos)` to flip `locked = FALSE` on every row.
  2. Partitions the input by `(mode, recovery_state)` and runs two batched balance flows — one for transparent rows, one for recovered shielded rows — through the same kind-aware `updateAddressLockedBalance` / `updateWalletLockedBalance` helpers used by the receive/spend paths. Unowned (or recovery-failed) shielded rows get their row-level `locked = FALSE` flip but no balance write; when the wallet later registers and recovery succeeds, the recovery path reads the row's current `locked = FALSE` and credits `unlocked_shielded_balance` directly. No follow-up unlock is needed.

**Why this is the central argument for the design.** A parallel-tables alternative would add a `…ShieldedOutputs(vertex)` companion call to every one of the four vertex handlers above *plus* a parallel `unlockShieldedTimelockedUtxos` sweep job, each performing the same arithmetic on parallel tables. Even with helpers that share an implementation, the *call sites* multiply, and every future change to the handlers would have to land in two places at once. With this design, the call sites stay the same; the **dispatch on `mode` lives inside the balance-update functions**, which is exactly the layer that already handles the per-row variance the transparent path requires (authority bits, locked vs unlocked, timelock expiry).

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
- `packages/daemon/src/db/index.ts` — `insertTxOutput`, `markTxOutputSpent`, `updateAddressTablesWithTx`, `updateWalletTablesWithTx` gain kind-awareness. New helpers `insertShieldedTxOutputData`, `upsertShieldedAddressObservation`, `findShieldedAddressOwnership`, `markTxOutputRecovered`, `markTxOutputRecoveryFailed` are added but live next to the existing helpers, not in a separate `shielded.ts` namespace.
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
- **Schema breadth.** Two new tables (`shielded_tx_output_data`, `shielded_address`) plus five modified tables. Modifications are additive; the migration is reversible by dropping the new columns and tables.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why one `tx_output` table instead of a parallel shielded outputs table?

The argument for **two physical tables** is isolation: a bug in shielded handling cannot corrupt rows the transparent code path reads, and shielded ingestion can be added without touching the hot `tx_output` table at all.

The argument for **one table** is handler simplicity: reorg, void, unvoid, vertex-removal, and input-consumption code paths are exactly the same arithmetic and don't need duplication. The point of `tx_output.mode` is to make the discrimination explicit in queries that care, and invisible in queries that don't (which is most of them).

This design takes the second trade-off. Reorg correctness is the most sensitive surface in the daemon; keeping one set of reorg/void/unvoid handlers — with kind dispatch pushed down into the balance-update layer — minimises what has to stay correct in lockstep. The risk that a bad shielded write corrupts transparent rows is real and is mitigated by test coverage of the dispatch logic (see § *Drawbacks*).

## Why a 1:1 satellite for crypto bytes instead of inlining them on `tx_output`?

Inlining would balloon every transparent row with NULL columns that consume disk. Range proofs and surjection proofs are large (~770–930 bytes of cryptographic material per shielded output). A 1:1 satellite keeps the hot `tx_output` table narrow.

The FK with `ON DELETE CASCADE` keeps lifecycle parity automatic: deleting a `tx_output` row on `handleVertexRemoved` removes the satellite row in the same statement.

## Why one `shielded_address` table with nullable ownership fields, mirroring transparent `address`?

The transparent `address` table already has this shape: rows exist for every observed address; `wallet_id` is NULL until a wallet that derives the address registers. Mirroring that pattern for shielded:

- Lets the daemon track activity (`transactions` counter) for spend_addresses that aren't yet claimed by any wallet.
- Makes wallet registration a per-row UPSERT against the existing pool, instead of an INSERT that might collide with what observation has written.
- Keeps the gap-extension behaviour at registration symmetric with the transparent path.

The cost is that ownership fields are nullable, which means queries that want owned-only rows must include `WHERE wallet_id IS NOT NULL`. This is a one-line filter; it doesn't multiply across many call sites.

## Why `address_balance.kind` as a label instead of in the PK?

Address spaces are disjoint — a given base58 string is either a transparent address or a shielded spend_address, never both. So `(address, token_id)` already uniquely identifies a row. Including `kind` in the PK would be redundant. Keeping it as a label column lets consumers filter by kind without JOINing through the ownership tables.

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
