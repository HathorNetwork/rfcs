- Feature Name: `wallet_service_shielded_outputs_daemon_schema`
- Start Date: 2026-05-04
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Add a parallel **shielded ingestion subsystem** to the Hathor `wallet-service` daemon and database. New, isolated tables (`shielded_tx_output`, `shielded_address`, `shielded_address_balance`, `shielded_wallet_balance`, `shielded_address_tx_history`, `shielded_wallet_tx_history`) hold all shielded-output state. The daemon's `SyncMachine` is extended with shielded substeps inside the existing per-vertex transaction: read `shielded_outputs[]` from the vertex event (a top-level sibling array delivered by the fullnode events API, not a `TxHeader` variant), persist every shielded output observed, look up ownership by `spend_partial_address`, and — on a match — perform an in-line range-proof rewind to recover `{ value, token_id }` and update the parallel balance tables. Reorg, void, and unvoid handling mirror the transparent path inside the same DB transaction. Push-notification and WebSocket update channels gain shielded-event variants.

# Motivation
[motivation]: #motivation

Hathor is introducing **shielded outputs** — a new output type that hides amount (and optionally token) using Pedersen commitments, Bulletproofs, and surjection proofs. The wallet-service is the canonical indexer for hosted Hathor wallets: it must index shielded outputs the same way it indexes transparent outputs, while keeping the existing transparent code path stable.

The challenge is that shielded outputs are physically very different from transparent outputs: rows are an order of magnitude larger (typical 770–930 bytes of cryptographic material), ownership is determined via a different mechanism (script-extracted `spend_partial_address` rather than a fullnode-pre-decoded address), and recovering the value requires CPU-bound cryptographic work. Bolting these into the existing `tx_output` table would balloon row sizes, branch every hot-path query on a new dimension, and couple the well-tested transparent flow to the experimental shielded flow.

The design isolates the shielded subsystem at the storage and code level so:

- A bug in shielded handling cannot regress transparent balances or reads.
- Shielded ingestion can be exercised by tests, profiled, or rolled back with no impact on the transparent path.
- The fullnode-side wire format (still in flux) is parsed in one place; a contract change is a localised edit.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A wallet-service contributor working on the daemon today thinks in terms of one ingestion pipeline: a vertex arrives, `handleVertexAccepted` walks `outputs[]`, and the result is rows in `tx_output`, `address_balance`, and `wallet_balance`.

After this RFC, there are **two parallel pipelines** — transparent and shielded — that run inside the same per-vertex DB transaction:

```
                      handleVertexAccepted (vertex v)
                                 │
            ┌────────────────────┴────────────────────┐
            ▼                                         ▼
   transparent substep                         shielded substep
   reads v.outputs[]                           reads v.shielded_outputs[]
   writes tx_output, address,                  writes shielded_tx_output,
   address_balance, wallet_balance,            shielded_address_balance,
   address_tx_history, wallet_tx_history       shielded_wallet_balance,
                                               shielded_address_tx_history,
                                               shielded_wallet_tx_history
```

Both substeps share the same input vertex and the same DB transaction. Either both succeed and the vertex is committed, or both roll back.

The shielded substep introduces three concepts the contributor must hold in their head:

1. **`spend_partial_address`** — the on-chain match key for shielded outputs. It is `hash(spend_pubkey)`, extracted from the P2PKH locking script inside each shielded output. It plays the same role for shielded ingestion that `output.decoded.address` plays for transparent ingestion. It is **not** the user-facing shielded address (which is the longer base58 string of the joined scan/spend pubkeys); it is just the on-chain match key.

2. **Eager rewind on match.** When a shielded output's `spend_partial_address` matches a row in `shielded_address`, the daemon synchronously calls `rewindAmountShieldedOutput` or `rewindFullShieldedOutput` (from `hathor-ct-crypto`) using the cached per-index `scan_privkey` for that row. On success, the recovered `{ value, token_id }` is written back to the `shielded_tx_output` row. The blinding factors **are not stored** — the user's wallet client re-derives them deterministically when constructing a spend.

3. **Parallel reorg/void.** Whenever the existing transparent path runs reorg, void, or unvoid logic on a vertex, the shielded substep does the same against the shielded tables. The patterns are identical (mark `voided`, reverse the running balance, append history rows); only the table names differ.

A contributor adding a new feature that needs to "iterate over a wallet's UTXOs" must now read **both** `tx_output` and `shielded_tx_output` and understand that they are two halves of the same ledger. The wallet-service helper layer (`packages/wallet-service/src/db/`) provides combined-read helpers so consumers do not have to remember this everywhere.

## Worked example: receive

User Alice's wallet has shielded keys registered. Bob sends Alice 1.5 HTR via an `AMOUNT_SHIELDED` output.

1. Fullnode emits `NEW_VERTEX_ACCEPTED` for the transaction. The vertex event has a top-level `shielded_outputs[]` array with one entry (deserialised by the events API from the on-chain `SHIELDED_OUTPUTS_HEADER`; the daemon does not see the raw header).
2. Daemon's `handleVertexAccepted` runs. The transparent substep finds nothing (`outputs[]` is empty in this example), so transparent tables are untouched.
3. The shielded substep parses the one shielded output. It extracts `spend_partial_address` from the embedded P2PKH script and inserts a `shielded_tx_output` row with all the on-chain bytes plus the parsed match key.
4. It looks up `spend_partial_address` in `shielded_address`. Alice's wallet has a row at `shielded_index = 7` with `spend_partial_address` matching. Hit.
5. It loads Alice's per-index `scan_privkey` and calls `rewindAmountShieldedOutput(scan_privkey, ephemeral_pubkey, commitment, range_proof, token_uid_from_token_data)`. Returns `{ value: 150, blindingFactor: ... }` (Hathor uses 2-decimal cents as its smallest unit, so `150` represents 1.50 HTR).
6. It updates the `shielded_tx_output` row with `recovered_value = 150`, `recovered_token_id = '00'` (HTR), and `recovery_state = 'recovered'`.
7. It upserts `shielded_address_balance` for `(spend_partial_address, '00')` and `shielded_wallet_balance` for `(wallet_id, '00')`, adding 1.5 HTR to `unlocked_balance`.
8. It appends `shielded_address_tx_history` and `shielded_wallet_tx_history` rows for the receive.
9. It enqueues a push notification and WebSocket update.
10. The DB transaction commits.

If the same vertex also had transparent outputs, those would be processed by the transparent substep in the same transaction.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Schema additions

All identifiers below are working names; final names are at the implementor's discretion. Types are written portably; engine-specific concerns are flagged.

### `shielded_tx_output`

Holds **every** shielded output observed on chain, regardless of ownership. This is the foundation that lets an upgrading wallet retroactively claim its historical receives.

```
shielded_tx_output
  tx_id                   VARCHAR(64)   NOT NULL,
  -- Concatenated index: position in (outputs ++ shielded_outputs). For a tx with
  -- N transparent outputs, the i-th shielded output has output_index = N + i.
  -- This is the same index used by inputs that consume the output.
  output_index            SMALLINT      NOT NULL,
  -- Mode and on-chain bytes.
  mode                    TINYINT       NOT NULL,  -- 1 = AMOUNT_SHIELDED, 2 = FULLY_SHIELDED
  commitment              VARBINARY(33) NOT NULL,
  range_proof             VARBINARY(1024) NOT NULL,
  script                  VARBINARY(1024) NOT NULL,
  ephemeral_pubkey        VARBINARY(33) NOT NULL,
  -- Mode-conditional fields.
  token_data              TINYINT       NULL,    -- non-null iff mode = AMOUNT_SHIELDED
  asset_commitment        VARBINARY(33) NULL,    -- non-null iff mode = FULLY_SHIELDED
  surjection_proof        VARBINARY(4096) NULL,  -- non-null iff mode = FULLY_SHIELDED
  -- Match key. Populated from output.decoded.address (delivered by the fullnode).
  -- This is hash(spend_pubkey); same byte shape as a transparent address hash.
  spend_partial_address   VARBINARY(20) NOT NULL,
  -- Recovery state, populated on match.
  recovery_state          ENUM('unowned','recovered','recovery_failed') NOT NULL DEFAULT 'unowned',
  recovered_value         BIGINT UNSIGNED NULL,
  recovered_token_id      VARCHAR(64)     NULL,    -- non-null iff recovered (32-byte UID, 'HTR' coded as zeros)
  -- Bookkeeping aligned with tx_output.
  spent_by                VARCHAR(64)   NULL,
  voided                  BOOLEAN       NOT NULL DEFAULT FALSE,
  first_block             VARCHAR(64)   NULL,
  timelock                INT UNSIGNED  NULL,
  heightlock              INT UNSIGNED  NULL,
  locked                  BOOLEAN       NOT NULL DEFAULT FALSE,
  created_at              TIMESTAMP     NOT NULL,
  updated_at              TIMESTAMP     NOT NULL,
  PRIMARY KEY (tx_id, output_index),
  INDEX idx_spend_partial_address (spend_partial_address),
  INDEX idx_voided_recovery (voided, recovery_state),
  INDEX idx_spent_by (spent_by)
```

Notes:

- `script` is stored raw for archival. The daemon does not parse it (`decoded.address` is supplied by the fullnode) but persisting the bytes lets future tooling re-analyse them if needed.
- `recovery_state` is an explicit enum rather than `recovered_value IS NULL`, because "unowned" and "recovery failed for an owned address" are operationally different states (the latter triggers an alert).
- The `spent_by` column is set by the input-handling pass when a transaction consumes this output; the dereference uses the same `(prev_tx_id, prev_index)` mechanism as transparent inputs (see § *Spend / consume tracking*).

### `shielded_address`

Holds wallet-owned shielded addresses. The on-chain match key is the primary key.

```
shielded_address
  spend_partial_address   VARBINARY(20)  NOT NULL,  -- PK; matches shielded_tx_output.spend_partial_address
  wallet_id               VARCHAR(64)    NOT NULL,
  shielded_index          INT UNSIGNED   NOT NULL,
  shielded_address        VARCHAR(100)   NOT NULL,  -- 71-byte payload, base58, ≤100 chars
  scan_privkey            VARBINARY(32)  NOT NULL,  -- pre-derived child scan privkey, stored in plaintext
  scan_pubkey             VARBINARY(33)  NOT NULL,
  spend_pubkey            VARBINARY(33)  NOT NULL,
  catchup_state           ENUM('pending','running','done') NOT NULL DEFAULT 'pending',
  transactions            INT UNSIGNED   NOT NULL DEFAULT 0,
  created_at              TIMESTAMP      NOT NULL,
  PRIMARY KEY (spend_partial_address),
  UNIQUE KEY uk_wallet_shielded_index (wallet_id, shielded_index),
  INDEX idx_shielded_address (shielded_address),
  INDEX idx_wallet_catchup (wallet_id, catchup_state)
```

Notes:

- `spend_partial_address` is PK because it is what the daemon hot path looks up. Lookup is one indexed equality on `VARBINARY(20)`.
- `shielded_address` (the long base58 string) is **also** indexed because the wallet API needs to resolve "is this string mine" for incoming pay-to-shielded-address requests.
- `scan_privkey` is stored in plaintext. Encryption-at-rest is out of scope for this design.
- `catchup_state` is the daemon's per-row marker for whether an upgrading wallet's historical re-scan has visited this address yet; it lets a long-running catch-up resume cleanly across restarts. It has no semantic meaning to the daemon's live ingest path. The full re-scan flow is described in design doc 2.
- The pair `(wallet_id, shielded_index)` is unique — the same wallet cannot register two shielded keys at the same BIP32 index.

### `shielded_address_balance` and `shielded_wallet_balance`

```
shielded_address_balance
  spend_partial_address   VARBINARY(20)   NOT NULL,
  token_id                VARCHAR(64)     NOT NULL,
  unlocked_balance        BIGINT UNSIGNED NOT NULL DEFAULT 0,
  locked_balance          BIGINT UNSIGNED NOT NULL DEFAULT 0,
  timelock_expires        INT UNSIGNED    NULL,
  transactions            INT UNSIGNED    NOT NULL DEFAULT 0,
  total_received          BIGINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (spend_partial_address, token_id)

shielded_wallet_balance
  wallet_id               VARCHAR(64)     NOT NULL,
  token_id                VARCHAR(64)     NOT NULL,
  unlocked_balance        BIGINT UNSIGNED NOT NULL DEFAULT 0,
  locked_balance          BIGINT UNSIGNED NOT NULL DEFAULT 0,
  timelock_expires        INT UNSIGNED    NULL,
  transactions            INT UNSIGNED    NOT NULL DEFAULT 0,
  total_received          BIGINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (wallet_id, token_id)
```

These mirror `address_balance` and `wallet_balance` shape-for-shape so the application-layer balance helpers can be near-copies. **Authority bits are intentionally absent** — shielded outputs cannot carry mint/melt authority. The upstream RFC keeps authority outputs transparent (Rule 7), and an in-flight follow-on RFC adds shielded *value* outputs to mint/melt transactions while keeping the authority outputs themselves transparent (see § *Future possibilities*).

### `shielded_address_tx_history` and `shielded_wallet_tx_history`

```
shielded_address_tx_history
  spend_partial_address   VARBINARY(20)   NOT NULL,
  tx_id                   VARCHAR(64)     NOT NULL,
  token_id                VARCHAR(64)     NOT NULL,
  balance_delta           BIGINT          NOT NULL,  -- signed
  timestamp               INT UNSIGNED    NOT NULL,
  voided                  BOOLEAN         NOT NULL DEFAULT FALSE,
  PRIMARY KEY (spend_partial_address, tx_id, token_id),
  INDEX idx_token_timestamp (token_id, timestamp DESC)

shielded_wallet_tx_history
  wallet_id               VARCHAR(64)     NOT NULL,
  tx_id                   VARCHAR(64)     NOT NULL,
  token_id                VARCHAR(64)     NOT NULL,
  balance_delta           BIGINT          NOT NULL,
  timestamp               INT UNSIGNED    NOT NULL,
  voided                  BOOLEAN         NOT NULL DEFAULT FALSE,
  PRIMARY KEY (wallet_id, tx_id, token_id),
  INDEX idx_token_timestamp (token_id, timestamp DESC)
```

Same shape as the transparent counterparts; voided rows are kept (not deleted) so reorg can be undone.

## Event ingestion

Confirmed wire format (fullnode-team alignment, 2026-05-04):

```ts
// packages/daemon/src/types/event.ts (additions)

export const ShieldedOutputModeSchema = z.union([
  z.literal('AMOUNT_SHIELDED'),
  z.literal('FULLY_SHIELDED'),
]);

// decoded.address is populated by the fullnode with hash(spend_pubkey) — same
// semantics as decoded.address on a transparent output.
const ShieldedDecodedSchema = z.object({
  address: z.string(),  // hex of the spend partial address
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
  mode: z.literal('AMOUNT_SHIELDED'),
  token_data: z.number().int(),
});

export const FullyShieldedOutputSchema = BaseShieldedOutputSchema.extend({
  mode: z.literal('FULLY_SHIELDED'),
  asset_commitment: z.string(),
  surjection_proof: z.string(),
});

export const ShieldedOutputSchema = z.discriminatedUnion('mode', [
  AmountShieldedOutputSchema,
  FullyShieldedOutputSchema,
]);

// Vertex event extension: a sibling top-level array, parallel to outputs[].
export const VertexEventSchema = ExistingVertexEventSchema.extend({
  shielded_outputs: z.array(ShieldedOutputSchema).default([]),
});
```

**No script parsing in the daemon.** Because the fullnode pre-extracts `decoded.address`, the daemon reads the spend partial address as a field, exactly as it does for transparent outputs today. The raw `script` is still persisted to `shielded_tx_output.script` for archival and future re-analysis, but no parser exists for it inside the wallet-service.

**Parsing isolation.** The schema lives in one module (`packages/daemon/src/types/event.ts`). The handler imports `shielded_outputs[]` from the parsed event and passes it to a shielded-specific service. A future change to the wire format is a single-file edit.

## Daemon `handleVertexAccepted` extension

Pseudocode for the substep added to `packages/daemon/src/services/index.ts`:

```ts
// ... existing transparent substep ...

if (vertex.shielded_outputs.length > 0) {
  await processShieldedOutputs(db, txn, vertex);
}

// ... rest of handleVertexAccepted ...
```

Where `processShieldedOutputs` is roughly:

```ts
async function processShieldedOutputs(db, txn, vertex) {
  const transparentCount = vertex.outputs.length;

  for (const [shieldedIdx, so] of vertex.shielded_outputs.entries()) {
    // The on-chain output_index is the concatenated index across (outputs ++ shielded_outputs).
    const outputIndex = transparentCount + shieldedIdx;

    // 1. Spend partial address comes directly from the fullnode-supplied decoded.address.
    const spendPartialAddress = Buffer.from(so.decoded.address, 'hex');

    // 2. Persist the on-chain bytes regardless of ownership.
    await db.insertShieldedTxOutput(vertex.tx_id, outputIndex, so, spendPartialAddress);

    // 3. Lookup ownership.
    const owned = await db.findShieldedAddressByPartial(spendPartialAddress);
    if (!owned) continue;

    // 4. Use the cached per-index scan privkey to rewind.
    let recovered;
    try {
      if (so.mode === 'AMOUNT_SHIELDED') {
        recovered = rewindAmountShieldedOutput(
          owned.scan_privkey, so.ephemeral_pubkey, so.commitment, so.range_proof,
          tokenUidFromTokenData(so.token_data, vertex),
        );
        await db.markShieldedRecovered(vertex.tx_id, outputIndex, recovered.value, recovered.tokenUid);
      } else {
        recovered = rewindFullShieldedOutput(
          owned.scan_privkey, so.ephemeral_pubkey, so.commitment, so.range_proof, so.asset_commitment,
        );
        verifyAssetCommitment(recovered, so.asset_commitment);  // mandatory per client guide §4.3
        await db.markShieldedRecovered(vertex.tx_id, outputIndex, recovered.value, recovered.tokenUid);
      }
    } catch (e) {
      // The output is owned (matched by hash) but rewind failed.
      // This is anomalous — it usually means malformed range_proof or wrong scan key.
      await db.markShieldedRecoveryFailed(vertex.tx_id, outputIndex);
      emitAlert('shielded_recovery_failed', { tx_id: vertex.tx_id, output_index: outputIndex });
      continue;
    }

    // 5. Update balance and history tables.
    await db.applyShieldedReceive(owned, vertex, recovered);
  }
}
```

`tokenUidFromTokenData` resolves the 1-byte `token_data` to its 32-byte UID by reading the vertex's `tokens[]` array (existing pattern; see `services/index.ts` for how transparent outputs already do this).

Critical invariant: `processShieldedOutputs` runs on the **same DB transaction** as the existing transparent substep. The existing per-vertex transaction wrapper in `andleVertexAccepted` is reused unchanged.

## Spend / consume tracking

A transaction input that consumes a shielded output uses the same wire format as a transparent input — `(prev_tx_id, prev_index)` — with `prev_index` indexing into the **concatenation** of the previous transaction's `outputs[]` and `shielded_outputs[]`, in that order. Concretely, if the previous tx has `N` transparent outputs and `M` shielded outputs:

| `prev_index` value | Refers to |
|--------------------|-----------|
| `0 .. N-1` | `prev_tx.outputs[prev_index]` (transparent) |
| `N .. N+M-1` | `prev_tx.shielded_outputs[prev_index - N]` |

Because the daemon stores `shielded_tx_output.output_index` as the concatenated index (see § *Schema additions*), the dereference is symmetric with the transparent path:

```ts
async function markInputSpent(db, input, consumingTxId) {
  // Try transparent first.
  const t = await db.markTxOutputSpent(input.prev_tx_id, input.prev_index, consumingTxId);
  if (t.affected === 1) return;

  // Fall through to shielded.
  const s = await db.markShieldedTxOutputSpent(input.prev_tx_id, input.prev_index, consumingTxId);
  if (s.affected === 0) {
    throw new Error(`input references non-existent output ${input.prev_tx_id}:${input.prev_index}`);
  }

  // If the spent shielded output is owned, reverse its balance contribution.
  const owned = await db.findOwnershipForShieldedOutput(input.prev_tx_id, input.prev_index);
  if (owned) {
    await db.applyShieldedSpend(owned, consumingTxId, /* sign = -1 */);
  }
}
```

The application order (transparent first, then shielded) is arbitrary because at most one of the two tables holds a row for any given `(prev_tx_id, prev_index)` — they live in disjoint index ranges within the same `prev_tx`. The fallthrough is a 1-row indexed lookup; cost is negligible.

**Reorg / void.** If a consuming tx is voided, `markShieldedTxOutputSpent` is reversed (clear `spent_by`) and `applyShieldedSpend` is reversed (re-add the balance), exactly mirroring the transparent path.

**Cross-tx consumption invariant.** The fullnode is the authoritative validator for "an input cannot reference a shielded output that doesn't exist." The wallet-service treats this as an integrity error (logged + alerted) rather than a correctness failure on its own.

## Reorg, void, unvoid

Each existing handler in `packages/daemon/src/services/index.ts` gains a shielded counterpart called from the same place inside the same transaction:

- `handleVertexRemoved` → also calls `removeShieldedOutputs(vertex)` to delete the corresponding `shielded_tx_output` rows and reverse balance deltas.
- `handleVoidedTx` → also calls `voidShieldedOutputs(vertex)` to set `voided = TRUE` on `shielded_tx_output` rows and reverse balance deltas in `shielded_address_balance` / `shielded_wallet_balance`. History rows are kept with `voided = TRUE` flag.
- `handleUnvoidedTx` → reverses the void: clears `voided`, re-applies balance deltas.
- `handleReorgStarted` → no shielded-specific logic at the start; reorgs are realised by the per-vertex remove/void calls that follow.

The patterns are exact copies of the transparent versions modulo table names. The DB layer offers helpers (`shieldedBalanceApply`, `shieldedBalanceReverse`) that take a sign argument so apply and reverse share an implementation.

## Push notifications and WebSocket updates

Two channels exist today:

- Push notifications (FCM via SQS) — `packages/daemon/src/services/pushNotification` and the alerts subsystem.
- WebSocket updates to connected wallet clients — `packages/wallet-service/src/ws/`.

For each, this design adds a single new event type, `'new-shielded-tx'`, payload-compatible with the existing `'new-tx'` event but carrying:

- `wallet_id`, `tx_id`
- The shielded slice of the tx: per-`spend_partial_address` per-token `balance_delta`, mode, output index.
- The transparent slice (if any) is **not** duplicated; the existing `'new-tx'` event handles that.

Clients receive both events when a tx affects both kinds of outputs. The daemon emits them in deterministic order: transparent first, then shielded, both inside the same per-vertex transaction's commit hook.

Push payloads omit any cryptographic material (commitments, ephemeral pubkeys); they carry only the user-visible summary (`amount`, `token_id`, `from_address` is intentionally **not** emitted because shielded outputs don't have a sender-revealing field).

## Mempool

Shielded outputs in mempool transactions are handled exactly as transparent mempool outputs are today. No special handling is added. The same `handleVertexAccepted`-equivalent path runs for mempool ingestion, and the same parallel substeps apply. This is called out explicitly because it was checked during design.

## Code organisation

- `packages/daemon/src/services/shielded/` — new directory holding `processShieldedOutputs`, `removeShieldedOutputs`, `voidShieldedOutputs`, `unvoidShieldedOutputs`, and the spend-side `markShieldedTxOutputSpent` / `applyShieldedSpend` helpers.
- `packages/daemon/src/db/shielded.ts` — DB helpers parallel to the existing functions in `db/index.ts` (e.g., `insertShieldedTxOutput`, `findShieldedAddressByPartial`, `applyShieldedReceive`, `markShieldedTxOutputSpent`).
- `packages/daemon/src/crypto/ctRewind.ts` — thin wrapper around the `hathor-ct-crypto` NAPI binding (centralises error handling and TypeScript types).
- `packages/common/src/types.ts` and `packages/daemon/src/types/event.ts` — extend the shared event/vertex schema types with the new top-level `shielded_outputs[]` field and its mode-discriminated entries. The existing `TxHeader` union is **not** modified; shielded outputs are not delivered through the header surface.

There is **no** script-parsing module: the fullnode delivers `decoded.address` per shielded output, so the daemon does not parse scripts. There is **no** encryption helper: scan-side secrets are stored in plaintext (encryption-at-rest is out of scope).

# Drawbacks
[drawbacks]: #drawbacks

- **CPU on the daemon.** Each ownership match triggers a ~1 ms range-proof rewind in-line. Match rate scales with the number of shielded-enabled wallets and the chain's shielded volume. The daemon is a long-running process with budget for this, but a sudden spike (e.g., an exchange registering many shielded wallets at once) could lengthen per-vertex processing. Mitigation: the rewind is per-output, not per-vertex, so backpressure is bounded by `outputs-per-tx × matches-per-output`.
- **Two reorg pipelines to keep correct.** Reorg correctness is already tested in the transparent path; we double the surface. Mitigation: the shielded helpers are near-textual copies of the transparent ones, and integration tests should exercise reorgs that involve both.
- **A second NAPI dependency** (`hathor-ct-crypto`) on the daemon. Mitigation: wrapped behind `crypto/ctRewind.ts` for centralised error handling.
- **Plaintext scan secrets in the DB.** Encryption-at-rest is out of scope. Mitigation: encryption can be added later without changing the schema layout — column types and widths leave room for ciphertext.
- **Schema breadth.** Six new tables. Migrations are additive and reversible.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why separate tables instead of a `kind` flag on `tx_output`?

Considered during early design and rejected. The alternative — extending `tx_output` with nullable shielded columns and a `kind` discriminator — was rejected because:

- 99% of historical rows are transparent and would carry a column overhead they never use.
- Every existing query against `tx_output` would need a `WHERE kind = 'transparent'` clause to keep working unchanged, multiplying review surface.
- A bug in shielded ingestion that wrote bad data could corrupt rows that the transparent code path also reads.

Separate tables means the only shared concept is the **vertex**, not the storage layer; there is no path by which shielded code can corrupt transparent data.

## Why synchronous in-line rewind rather than async?

Considered during early design and rejected. Async post-processing fell out because:

- It introduces an inconsistency window between transparent and shielded views of the same tx, which complicates the API contract ("is this balance up-to-date?").
- It complicates reorg ordering: an undo job for vertex N must not overtake an apply job for vertex N+1.
- The rewind cost (~1 ms per matched output) is small relative to the I/O work the daemon already does per vertex.

The match step itself is O(1) on an indexed key, so unowned outputs (the common case) cost essentially nothing.

## Why per-index pre-derived scan privkeys instead of HD-derive on demand?

The wallet-service has two execution environments (daemon, Lambda API). Lambda CPU per request is cost-sensitive and HD child derivation is CPU-bound. Pre-deriving once at gap-limit-extension time lets the API read a cheap indexed row, rather than running HD math on every request. The daemon could derive on demand cheaply, but consistency of behaviour across both environments is operationally simpler if everything reads from the same column. (The choice also leaves the schema unchanged when encryption-at-rest lands later: the column type and lifecycle are the same; only the on-disk encoding changes.)

## Why store `recovered_value` / `recovered_token_id` on `shielded_tx_output` rather than in a separate "recovery" table?

A separate table was considered. Rejected because (a) recovery is one-to-one with the shielded output and never moves, (b) it would force every read that wants both raw and recovered data to JOIN, and (c) the recovery columns are only populated when `recovery_state = 'recovered'`, so they're cheap NULL columns when unowned.

## Why keep history tables instead of computing history from `shielded_tx_output`?

Symmetry with transparent. The transparent path has `address_tx_history` and `wallet_tx_history` precisely because reconstructing a paginated wallet history from `tx_output` joins is too slow at scale; the same arithmetic applies to the shielded subsystem.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The wire-format and library-stability questions raised during early design have been resolved with the fullnode team. The remaining items are local design choices:

1. **Should `recovered_value` be re-checked when `total_received` is incremented?** I.e., do we ever expect to amend a recovery? Probably no, but the column is mutable in the schema as written. Decision: keep mutable; document that the daemon never overwrites a `recovery_state = 'recovered'` row.
2. **Behaviour on `recovery_failed`.** Today's design alerts and continues. An alternative is to halt the daemon (treat as a consensus error). The first behaviour is preferred because a malformed proof would have been rejected by the fullnode upstream; a recovery failure here implies a key/derivation bug, not a data-corruption bug, and halting would be over-broad.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Background batch rewind for catch-up.** When an existing wallet upgrades (the catch-up flow described in design doc 2), the catch-up scan is naturally async and parallel; this design's synchronous-on-the-hot-path choice does not preclude an out-of-band batch worker, it merely declines to use one for the main pipeline.
- **Indexing for view-key delegation.** If the upstream RFC's "view key delegation" feature lands, the indexer is naturally positioned to act as the delegated viewer. The schema accommodates this (the per-row `scan_privkey` is already the only secret needed) without further migrations.
- **Nullifier-based consumption tracking.** If phase C of the upstream RFC lands nullifiers, `shielded_tx_output` gains an indexed `nullifier` column and the input-parser changes; the rest of the design is unaffected.
- **Range-proof verification by the indexer.** Currently delegated to the fullnode. If we ever want defense-in-depth, the rewind path can call `verify_range_proof` after rewind for a small extra cost.
- **Shielded mint/melt support.** An in-flight upstream RFC ([`feat/shielded-outputs/text/0000-shielded-outputs-mint-melt.md`](https://github.com/HathorNetwork/rfcs/blob/feat/shielded-outputs/text/0000-shielded-outputs-mint-melt.md)) adds shielded *value* outputs to mint/melt transactions while keeping authority outputs transparent (parent-RFC Rule 7 stays). It repeals Rule 8 (the prohibition on mixing shielded outputs with mint/melt) and introduces two new vertex headers — `MintHeader` (`0x14`) and `MeltHeader` (`0x15`), each carrying a list of `(token_index, amount)` entries that publicly declare per-token supply deltas while keeping recipient sets and per-recipient amounts private. Wallet-service impact when this lands:
  - Daemon parser extended to recognise `0x14` and `0x15`. The events API will surface mint/melt entries on the vertex event using the same approach already used for `shielded_outputs[]`.
  - Per-token supply tracking updated to apply the public mint/melt scalars from these headers, mirroring how transparent mint/melt is tracked today.
  - Existing shielded recovery flow handles incoming shielded value outputs from mint/melt transactions with no change — they look like any other `AmountShieldedOutput` or `FullShieldedOutput` to the rewind path.
  - Token-creation transactions become eligible to carry shielded outputs (under refined Rule M2); the existing token-creation indexing path is extended to call into the shielded substep when a `shielded_outputs[]` array is present on a TCT.
  - No schema change required for `shielded_tx_output` itself; the new work is parsing two new headers and updating supply counters.
