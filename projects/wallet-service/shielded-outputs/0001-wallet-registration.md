- Feature Name: `wallet_service_shielded_outputs_registration`
- Start Date: 2026-05-12
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Extend the wallet-service registration flow so a wallet can attach **scan and spend keys for shielded outputs** either at creation or later. New wallets register with `{ xpubkey, auth_xpubkey, scan_xpriv, spend_xpub }` in a single call; existing transparent-only wallets attach the shielded keys later through the same load endpoint, which triggers a historical re-scan. Shielded address derivation is BIP32 with a gap-limit policy (default 20, matching transparent), driven by **two paths in lockstep** — `m/44'/280'/1'/0/{i}` (scan) and the matching spend path at the same `index`. The scan HD privkey and every per-index `scan_privkey` are stored **in plaintext**; encryption-at-rest is out of scope and described in § *Future possibilities*.

Wallet registration interacts with the unified `address` table defined in [`0000-daemon-and-database.md`](0000-daemon-and-database.md). Shielded ownership rows live on that same table with `bip32_account = 1`. The daemon writes a row on every observed shielded spend_address (with NULL ownership fields), so registration is an UPSERT that **fills in** the ownership fields on existing rows and inserts new rows where the wallet's derived spend_addresses have not been seen on-chain yet.

# Motivation
[motivation]: #motivation

The [daemon and database design](0000-daemon-and-database.md) establishes that the daemon ingests every shielded output observed on chain and matches owned ones via the `address` column on `tx_output` against the unified `address` table — filtering on `bip32_account = 1` and `wallet_id IS NOT NULL`. That match table has to be populated by *something*. This document specifies that "something":

- The API surface a wallet uses to declare its shielded keys (at creation or upgrade).
- The on-server derivation that turns those keys into ownership rows on the unified `address` table at `bip32_account = 1`.
- The historical re-scan that lets an upgrading wallet retroactively claim the shielded outputs it received before it was registered as shielded-enabled.

A second motivation is operational: the wallet-service runs a Lambda-deployed API where CPU per request is cost-sensitive. Doing BIP32 child derivation on every API call is significantly more expensive than reading a cached row. Pre-derivation at gap-limit-extension time keeps the read path cheap.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The wallet-service has always known one kind of wallet: a wallet identified by an xpub, with a gap-limited list of derived addresses. After this RFC, a wallet still has exactly **one identity** (one `wallet` row), but it can now also carry a **shielded layer**: an HD scan privkey, an HD spend xpub, and a derivation index it shares between them.

A developer adding shielded support to a wallet client thinks in terms of one operation: **initialise (or re-initialise) the wallet.** Call the existing wallet-load endpoint with whatever keys the client has — transparent xpub always, optionally `scan_xpriv` and `spend_xpub`. The server reconciles its stored state with the submitted keys: a new wallet is created, a transparent-only wallet that just received its first shielded keys transitions to a catch-up phase, an already-shielded wallet whose keys match is a no-op. The client never has to introspect server state to decide which endpoint to call — it always calls one endpoint with everything it has.

Address gap-extension is **not** a client-driven operation. It happens automatically server-side, exactly as it does for transparent today: as the daemon processes shielded outputs and `last_used_shielded_index` advances, the daemon derives more shielded `address` rows (`bip32_account = 1`) to maintain `shielded_max_gap` consecutive unused trailing addresses. The client never asks to extend the gap; it just asks for the next available shielded address (mirroring `GET /addresses/new` on the transparent side).

The contributor must hold three concepts:

- **Shielded index.** A single integer that drives **both** scan and spend derivations at the same position. There is no scenario in this design where the index differs between scan and spend; they advance together. On the unified `address` table this is the `index` column on rows with `bip32_account = 1`.
- **Pre-derivation as UPSERT.** When the API generates an ownership row for shielded index N, the daemon (or the API on registration) computes `spend_address` (the on-chain base58 spend address) for that index and runs an `INSERT … ON DUPLICATE KEY UPDATE` against the unified `address` table. If the row exists with NULL ownership (the daemon observed the spend_address on-chain before the wallet claimed it), the UPDATE fills in `wallet_id`, `index`, `bip32_account = 1`, `ct_address` (long-form CT address), `scan_privkey`, `catchup_state`. If the row does not exist, the INSERT creates it. Either way, the row is ready for the daemon's hot path. `scan_pubkey` and `spend_pubkey` are computed as intermediates during derivation but are not stored — they are recoverable on demand from `scan_privkey` (point multiplication) and `wallet.spend_xpub` (BIP32 derivation).
- **Catch-up scan.** A bounded job that runs once per upgrade or gap-extension. It is **idempotent** (re-running it does not double-credit balances) so it can be retried safely on failure.

## Worked example: client adds shielded support

User Alice has a transparent-only wallet `wallet_alice` registered three months ago. She updates her wallet client, which now supports shielded outputs. The client does **not** know whether the wallet-service already has its shielded keys — it just calls the existing wallet-load endpoint (`POST wallet/init`, handler `wallet.load`) with everything it has, including the new `scan_xpriv` and `spend_xpub` derived from her seed plus signatures over them:

```
POST /wallet/init
{
  "xpubkey":              "...",
  "authXpubkey":          "...",
  "xpubkeySignature":     "...",
  "authXpubkeySignature": "...",
  "timestamp":            1746816000,
  "firstAddress":         "...",

  "scanXpriv":            "...",
  "spendXpub":            "...",
  "scanXprivSignature":   "...",
  "spendXpubSignature":   "..."
}
```

The shielded fields follow the same flat shape as the existing transparent signature fields — no nested `signatures` object, just sibling keys. All four shielded fields are all-or-none.

The server reconciles. It looks up `wallet_alice` and finds it exists with `status = 'ready'`, `shielded_status = 'none'`. It then:

1. Validates the existing transparent signatures (unchanged path).
2. Validates `scanXprivSignature` and `spendXpubSignature` against `auth_xpubkey/0` to confirm the caller has authority over both new keys.
3. Confirms the `wallet` row has no `scan_xpriv` stored (this is the upgrade case, not a key-mismatch case).
4. Sets `wallet.scan_xpriv`, `wallet.spend_xpub`, `shielded_max_gap = 20`, `shielded_status = 'catching-up'`.
5. Pre-derives 20 shielded addresses (indexes 0..19). For each, runs `INSERT … ON DUPLICATE KEY UPDATE` against the unified `address` table to either fill in ownership on a row the daemon had already observed, or insert a new owned row with `bip32_account = 1`. (`scan_pubkey` and `spend_pubkey` are computed during derivation but not stored.) New ownership rows are written with `catchup_state = 'pending'`.
6. Enqueues a catch-up job for `wallet_alice`.
7. Returns 200 with the wallet's status payload, including `shielded_status: 'catching-up'`. The client then polls the existing wallet-status endpoint until `shielded_status` reaches `'ready'`.

The catch-up job (running on the daemon, see § *Re-scan procedure*):

1. Reads the spend_addresses just claimed by `wallet_alice` (rows on `address` with `bip32_account = 1` and `catchup_state = 'pending'`).
2. Issues a paginated query against `tx_output` joined to `shielded_tx_output_data`:
   ```sql
   SELECT t.tx_id, t.index, t.mode, t.address, t.token_id,
          d.commitment, d.range_proof, d.ephemeral_pubkey,
          d.token_data, d.asset_commitment, d.surjection_proof,
          t.voided, t.first_block, a.scan_privkey
   FROM tx_output t
   INNER JOIN shielded_tx_output_data d
     ON (d.tx_id, d.index) = (t.tx_id, t.index)
   INNER JOIN address a ON a.address = t.address AND a.bip32_account = 1
   WHERE a.wallet_id = :wallet_id
     AND a.catchup_state = 'pending'
     AND t.mode IN (1, 2)
     AND t.voided = FALSE
     AND t.recovery_state = 'unowned'
   ORDER BY t.first_block ASC, t.tx_id ASC
   LIMIT :M
   ```
3. For each row, runs the same recovery logic as the live ingest path (rewind, validate, `UPDATE tx_output SET value=…, token_id=…, recovery_state='recovered' WHERE … AND recovery_state='unowned'`), then calls `updateAddressTablesWithTx`/`updateWalletTablesWithTx` to apply the recovered amount to `address_balance` / `wallet_balance` / `wallet_tx_history` with the appropriate columns.
4. Emits a `'shielded-catchup-progress'` event per page so the wallet client can render progress.
5. On completion, transitions `shielded_status` to `'ready'` and `address.catchup_state` to `'done'` for the processed addresses.

If the catch-up surfaces shielded receives at high indices, the catch-up algorithm itself extends the gap (see § *Re-scan procedure* — the algorithm is a fixpoint loop that repeats with newly-derived addresses until `shielded_max_gap` consecutive unused trailing addresses remain). Once Alice's wallet is `shielded_status = 'ready'`, ongoing gap-extension during normal ingest mirrors the transparent path: as the daemon processes a new shielded output that uses a high-index address, it derives more shielded `address` ownership rows (`bip32_account = 1`) to keep the gap stable.

### Variant: completely new wallet with shielded support

The client calls the same load endpoint with the full set of keys. The server finds no existing `wallet_alice`, runs the existing transparent creation path (sets `status = 'creating'`), and **in the same atomic step** sets `shielded_status = 'catching-up'`, derives the initial 20 shielded ownership rows on the unified `address` table at `bip32_account = 1` (each via the UPSERT described above), and enqueues both the transparent and shielded async initialisers. Both halves run in parallel. The client polls until both `status = 'ready'` and `shielded_status = 'ready'`.

### Variant: client without shielded support

The client calls the same load endpoint with only the transparent fields. The server runs the existing flow unchanged. Whether or not the wallet has shielded keys stored already, the absence of `scanXpriv` / `spendXpub` in the request is treated as "the client did not provide shielded data" — never as a request to disable shielded.

### Variant: shielded keys re-submitted matching what's stored

No-op. The server's reconciliation step finds `wallet.scan_xpriv` already populated and equal to the submitted value. It returns the current wallet status as it would for a transparent-already-loaded request.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Schema additions on `wallet`

New columns, all nullable to support the upgrade flow:

```sql
ALTER TABLE wallet ADD COLUMN scan_xpriv               VARBINARY(112) NULL;  -- HD privkey serialised, stored in plaintext
ALTER TABLE wallet ADD COLUMN spend_xpub               VARCHAR(120)   NULL;
ALTER TABLE wallet ADD COLUMN shielded_max_gap         SMALLINT UNSIGNED NULL DEFAULT 20;
ALTER TABLE wallet ADD COLUMN last_used_shielded_index INT UNSIGNED   NULL;
ALTER TABLE wallet ADD COLUMN shielded_status          ENUM('none','catching-up','ready','error') NOT NULL DEFAULT 'none';
```

`shielded_status` is independent of the existing `wallet.status` so the transparent and shielded views can be in different lifecycle phases at the same time (e.g., a wallet may be `status = 'ready'` for transparent while `shielded_status = 'catching-up'`).

`scan_xpriv` stores the BIP32-serialised extended private key as bytes in plaintext. The column type and width leave room for a future encryption-at-rest migration to be a column-data change, not a schema change.

## API additions

### Wallet-load endpoint — extended

The existing endpoint that creates-or-resumes a wallet (handler `wallet.load` in `packages/wallet-service/src/api/wallet.ts`, mapped to `POST wallet/init` in `serverless.yml`) is **the only entry point** for attaching shielded keys. There is **no** separate upgrade endpoint and no `/{wallet_id}/...` path segment — wallet identity is established by the body's `xpubkey` for the load endpoint and by the bearer authorizer for every other endpoint. The existing fields stay; four optional fields are added:

```jsonc
{
  "xpubkey":              "...",
  "authXpubkey":          "...",
  "xpubkeySignature":     "...",
  "authXpubkeySignature": "...",
  "timestamp":            1746816000,
  "firstAddress":         "...",

  // Optional. All-or-none — if any is provided, all must be.
  "scanXpriv":            "...",   // BIP32 HD privkey at m/44'/280'/1'
  "spendXpub":            "...",   // BIP32 HD xpub  at the matching shielded-spend account
  "scanXprivSignature":   "...",   // signature over `scanXpriv || timestamp` by authXpubkey/0
  "spendXpubSignature":   "..."    // signature over `spendXpub || timestamp` by authXpubkey/0
}
```

Validation of the shielded fields:

- All-or-none: if any of `scanXpriv`, `spendXpub`, `scanXprivSignature`, `spendXpubSignature` is present, all must be (`400 Bad Request` otherwise).
- Both signatures verified against `authXpubkey/0` using the existing `verifySignature` helper.
- Server confirms the spend xpub is a real BIP32 xpub (parses without error). No equivalent check on `scanXpriv` is necessary beyond signature validation.

#### Reconciliation table

The handler reconciles its stored state against the submitted body. The matrix is:

| Wallet exists? | `wallet.scan_xpriv` already stored? | Shielded fields in request? | Server behaviour |
|---|---|---|---|
| No | n/a | No | Existing transparent creation path. `shielded_status = 'none'`. |
| No | n/a | Yes | Transparent creation path **and** shielded init in the same atomic step: set `scan_xpriv`, `spend_xpub`, `shielded_max_gap = 20`; pre-derive 20 shielded ownership rows via UPSERT against the unified `address` table; `shielded_status = 'catching-up'`; enqueue catch-up job. |
| Yes | No | No | Existing transparent already-loaded behaviour: return current status. |
| Yes | No | Yes | **Upgrade case.** Verify shielded signatures; set `scan_xpriv`, `spend_xpub`, `shielded_max_gap = 20`; pre-derive 20 shielded ownership rows via UPSERT against the unified `address` table; `shielded_status = 'catching-up'`; enqueue catch-up job; return current status. |
| Yes | Yes (same as submitted) | Yes | No-op for the shielded half. Return current status. |
| Yes | Yes (different from submitted) | Yes | **Reject (`409 Conflict`).** Silently overwriting registered shielded keys is a security violation. |
| Yes | Yes | No | No-op. The client did not provide shielded data; this is **never** treated as a request to disable shielded. |

#### Errors

- `400` — shielded fields partially present (e.g., `scanXpriv` without `spendXpub`).
- `403` — `scanXprivSignature` or `spendXpubSignature` validation failed.
- `409` — submitted shielded keys differ from those already stored on the wallet.

#### Response

The endpoint returns the existing wallet-status payload — same wrapping (`{ "success": true, "status": { ... } }`), same shape returned today. The status object gains three new fields:

```jsonc
{
  "success": true,
  "status": {
    // ... existing fields (id, xpubkey, status, max_gap, last_used_address_index, etc.) ...
    "shielded_status":          "none" | "catching-up" | "ready" | "error",
    "shielded_max_gap":         20,
    "last_used_shielded_index": 7
  }
}
```

Clients poll the existing `GET wallet/status` endpoint to observe both `status` and `shielded_status` advancing. Old clients ignore the three new fields without issue (additive change).

### Address gap-extension

There is **no** client-facing gap-extension endpoint. The transparent path doesn't have one (gap-extension is a daemon-side side-effect of `addNewAddresses` in `packages/daemon/src/services/index.ts` whenever `last_used_address_index` advances within `max_gap` of the highest-derived index), and the shielded path mirrors it.

Concretely: when the daemon's shielded ingest flow recovers an output at shielded `index = N`, it updates `wallet.last_used_shielded_index = max(N, current)` and, if `last_used_shielded_index + shielded_max_gap > highest_derived_shielded_index`, derives additional ownership rows in the same DB transaction via UPSERT against the unified `address` table (`bip32_account = 1`). The reused helper is the `generateAddresses` library function (already used for transparent), parametrised with the spend xpub and scan xpriv.

The client requests "the next available shielded address" via the same endpoint that exists for transparent (`GET wallet/addresses/new`), extended to take an optional `kind=shielded` query parameter. That endpoint never derives — it only reads from the pool the daemon maintains.

## Address derivation

For each new shielded index `i`:

```ts
const scanXpriv  = wallet.scan_xpriv;   // plaintext
const spendXpub  = wallet.spend_xpub;

const scanChild  = bip32.derive(scanXpriv, `0/${i}`);   // child of m/44'/280'/1'
const spendChild = bip32.derive(spendXpub,  `0/${i}`);  // child of the matching shielded-spend account

const scanPriv   = scanChild.privateKey;     // 32 bytes
const scanPub    = scanChild.publicKey;      // 33 bytes compressed (intermediate; not stored)
const spendPub   = spendChild.publicKey;     // 33 bytes compressed (intermediate; not stored)

// On-chain spend address: base58(network_byte || hash160(spendPub) || checksum) — the match key.
const spendAddress = encodeSpendAddress(spendPub);            // base58, ≤34 chars
const ctAddress    = encodeCtAddress(scanPub, spendPub);      // long-form CT address, base58, ≤100 chars

await db.upsertShieldedAddressOwnership({
  address:       spendAddress,
  wallet_id:     wallet.id,
  index:         i,
  ct_address:    ctAddress,
  scan_privkey:  scanPriv,
  catchup_state: 'pending',
});
```

`upsertShieldedAddressOwnership` writes to the unified `address` table:

```sql
INSERT INTO address (address, bip32_account, wallet_id, `index`, ct_address, scan_privkey, catchup_state)
VALUES (?, 1, ?, ?, ?, ?, 'pending')
ON DUPLICATE KEY UPDATE
  bip32_account = 1,
  wallet_id     = VALUES(wallet_id),
  `index`       = VALUES(`index`),
  ct_address    = VALUES(ct_address),
  scan_privkey  = VALUES(scan_privkey),
  catchup_state = COALESCE(catchup_state, 'pending')
```

The registration path is the only writer of `ct_address` — the daemon's observation upsert leaves that column NULL because the long-form cannot be derived from the on-chain `spend_address` alone (it requires `scan_pubkey` + `spend_pubkey`, which only the registration path has at hand). So this UPSERT can safely write `VALUES(ct_address)` directly; there is no prior daemon-set value to preserve. The COALESCE on `catchup_state` is defensive: if for any reason the row already has `'done'` (e.g., a previous registration that was rolled back at a higher layer), we don't accidentally reset it to `'pending'` and re-run catch-up.

`encodeCtAddress` produces the user-facing string:

```
base58( network_byte(1B) || scan_pubkey(33B) || spend_pubkey(33B) || checksum(4B) )
```

Where `checksum = first_4_bytes(SHA256(SHA256(network_byte || scan_pubkey || spend_pubkey)))`, mirroring the transparent address checksum pattern.

The `network_byte` distinguishes shielded mainnet, shielded testnet, and any future network discriminators. Concrete byte values are out of scope here; they belong with the on-chain address-format specification.

## Re-scan procedure

The catch-up job runs on the daemon (not in the Lambda API) because it is unbounded in duration and must not block API request capacity. The single trigger is the wallet-load endpoint reconciling a wallet that just received its first shielded keys (the upgrade case, or the new-shielded-wallet case). Catch-up starts with the freshly-derived initial gap of `shielded_max_gap` ownership rows (default 20).

Catch-up must respect the BIP32 gap-limit invariant: it cannot stop until there are `shielded_max_gap` consecutive unused trailing shielded addresses. If the initial gap surfaces high-index receives, the algorithm derives more addresses and continues. This mirrors the transparent restoration loop in `generateAddresses` (`do { … } while (lastUsedAddressIndex + maxGap > highestCheckedIndex)`).

Job algorithm (idempotent, fixpoint loop):

```
loop:
  1. Read the candidate ownership set:
       SELECT a.address, a.`index`, a.scan_privkey
       FROM address a
       WHERE a.bip32_account = 1
         AND a.wallet_id = :wallet_id
         AND a.catchup_state = 'pending'

     If empty → goto step 4.

  2. Process each (address, scan_privkey) in batches of M:
       a. SELECT t.tx_id, t.index, t.mode, t.token_id,
                 d.ephemeral_pubkey, d.commitment, d.range_proof,
                 d.token_data, d.asset_commitment, t.first_block
          FROM tx_output t
          INNER JOIN shielded_tx_output_data d
            ON (d.tx_id, d.index) = (t.tx_id, t.index)
          WHERE t.address IN (...candidate addresses...)
            AND t.mode IN (1, 2)
            AND t.voided = FALSE
            AND t.recovery_state = 'unowned'
          ORDER BY t.first_block ASC, t.tx_id ASC
          LIMIT M

       b. For each row:
            - rewindAmount/FullShieldedOutput
            - on success: UPDATE tx_output SET recovery_state='recovered',
                          value=..., token_id=...
                          WHERE tx_id=... AND index=...
                          AND recovery_state='unowned'
            - apply to address_balance / wallet_balance / wallet_tx_history
              via updateAddressTablesWithTx/updateWalletTablesWithTx (the same
              helpers used by the live ingest path, dispatching on mode).
            - if this row matched a higher-index address than the wallet's current
              last_used_shielded_index, advance last_used_shielded_index.

       c. Mark progress: UPDATE address SET catchup_state='done'
          WHERE address IN (...processed addresses...)
            AND bip32_account = 1 AND wallet_id = :wallet_id

  3. Gap-extension check (the fixpoint step):
       last_used   = wallet.last_used_shielded_index   (-1 if no address has received yet)
       highest     = max(address.`index`) for this wallet WHERE bip32_account = 1
       max_gap     = wallet.shielded_max_gap

       If last_used + max_gap > highest:
         - Derive shielded address ownership rows for indexes (highest+1)..(last_used+max_gap),
           inserting via the UPSERT helper (writes to `address` with bip32_account = 1
           and catchup_state = 'pending').
         - Update wallet.last_used_shielded_index if last_used was advanced in step 2.
         - goto step 1 — the loop will scan the freshly-derived addresses.

       Else (gap is satisfied):
         - goto step 4.

  4. Catch-up complete:
       UPDATE wallet SET shielded_status='ready' WHERE id=:wallet_id

  5. Emit a final 'shielded-catchup-complete' event.
```

Termination is guaranteed: each loop iteration either marks at least one address as `done` (decrementing the pending set) or reaches the gap-satisfied condition. The chain has finite history, so the highest used shielded `index` for any wallet is bounded.

The `WHERE recovery_state = 'unowned'` filter on the row update makes the recovery write idempotent: a retry of the same output is a no-op.

The balance update is **not** idempotent on its own — the algorithm relies on the `recovery_state` transition guard to ensure a balance delta is applied at most once per `(tx_id, index)`.

Pagination size `M` is configurable; a reasonable default is 200 (~200 ms per batch at ~1 ms per rewind on commodity hardware).

`catchup_state` is a column on the unified `address` table (defined in the [daemon and database design](0000-daemon-and-database.md) § *Schema changes*); it lets the job resume cleanly across restarts. It is NULL for transparent rows (`bip32_account = 0`) and populated only for shielded rows. Newly-derived ownership rows from the gap-extension step are inserted with `catchup_state = 'pending'`, so a job that crashes between iterations resumes cleanly on the next run.

### Re-scan ordering and reorgs

The catch-up job reads `tx_output` and `shielded_tx_output_data` rows that the live ingest path is also writing to. Two interaction concerns:

- **Live writes during catch-up.** A new vertex containing a shielded output for a catching-up wallet is processed by the live path in the normal way (it sees the wallet's ownership row on `address` with `bip32_account = 1` and recovers in-line). The catch-up job will then also see that row, but the `recovery_state = 'unowned'` filter excludes already-recovered outputs. No double credit.
- **Reorgs during catch-up.** A reorg that voids a tx the catch-up job already credited reverses the balance through the unified void path (which dispatches on `tx_output.mode` to reverse the shielded balance columns). The catch-up job does not need special reorg handling.

## Storage of scan-side secrets

Scan-side secrets are stored in **plaintext**:

- `wallet.scan_xpriv` — the HD privkey for the scan path. Used by the API at gap-extension time to derive new children.
- `address.scan_privkey` (on `bip32_account = 1` rows) — the per-index child privkey. Used by the daemon on every match (hot path).

Encryption-at-rest is out of scope for this design and described in § *Future possibilities* below. The schema is shaped so a future encryption migration is a column-data change (re-encode bytes in place) and not a structural change.

### Why pre-derived children, even without encryption?

The argument for pre-derivation does not depend on encryption: HD child derivation is CPU-bound, the API runs on Lambda where per-request CPU is cost-sensitive, and the rate of API calls is much higher than the rate of gap-extension. Storing pre-derived child privkeys keeps the API read path cheap regardless of whether the column is encrypted.

## Authentication and authorisation

Submitting `scan_xpriv` to the wallet-service is a high-value operation: it surrenders read access to all current and future shielded receives for that wallet. A wallet-load request that includes shielded fields must carry both the existing transparent signatures (`xpubkeySignature` and `authXpubkeySignature`, proving authority over `auth_xpubkey`) **and** the shielded signatures (`scanXprivSignature` and `spendXpubSignature`, proving the same authority signed off on these specific shielded keys).

`scanXprivSignature` and `spendXpubSignature` are signatures by `authXpubkey/0` over `scanXpriv || timestamp` and `spendXpub || timestamp` respectively. The server verifies both before accepting the shielded portion of the load. A request that supplies shielded fields without valid signatures is rejected with `403`, regardless of whether the transparent half would otherwise have succeeded.

Rate-limit: at most one shielded-key reconciliation per wallet per hour (configurable; ops decision). A re-submission of identical keys within the limit is the no-op described in the reconciliation table — it does not re-run the catch-up job.

### Catch-up failure handling

If the async catch-up encounters an unrecoverable error (e.g., a stuck rewind, persistent DB error), the job sets `shielded_status = 'error'` on the wallet. `wallet.status` is unchanged — the transparent side stays whatever it was. A subsequent wallet-load call with the same shielded keys retries: the handler observes `shielded_status = 'error'`, transitions it back to `'catching-up'`, and re-enqueues the job. The catch-up algorithm's idempotency (the `recovery_state = 'unowned'` row guard described in § *Re-scan procedure*) makes the retry safe.

This behaviour is provisional pending ops-team review.

# Drawbacks
[drawbacks]: #drawbacks

- **Catch-up duration is unpredictable.** Worst case for a wallet with `shielded_max_gap = 20` upgrading after a year of heavy network shielded usage is bounded by `20 × O(unowned shielded outputs scanned)`. The bound is acceptable today (low shielded volume), but will need monitoring as the network grows. Mitigation: catch-up is paginated, restartable, and emits progress; a long catch-up does not block the API.
- **Plaintext scan secrets in the DB.** Anyone with read access to the `wallet` table and the shielded rows of `address` (`bip32_account = 1`) sees scan-side material in the clear, and can replay rewinds against any shielded output to learn amounts and tokens for any wallet whose secrets they have. Mitigation: the existing DB access-control posture (network isolation, IAM-scoped credentials) is the operative control. Encryption-at-rest is described in § *Future possibilities* as the next step.
- **More secret material in the DB.** Per-index `scan_privkey` is a row count proportional to `wallets × gap_limit`. At 10k wallets × 20 = 200k shielded rows on the `address` table, each carrying ~32 bytes of scan privkey. Negligible in storage, but more secret-material rows to govern. Mitigation: same DB access-control as above.
- **Two near-parallel signature-check paths** (transparent registration vs. shielded upgrade). Mitigation: share the `validateSignatures` helper.
- **Status surface for clients widens.** The shielded `catching-up` state is a new lifecycle phase clients must render. Mitigation: documented in the API spec; default rendering is "available, syncing".

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why UPSERT on `address` instead of INSERT-only?

The unified `address` table already contains rows for every spend_address the daemon has observed on-chain — including shielded ones (`bip32_account = 1`) with NULL ownership fields. When a wallet registers, its derived spend_addresses **may already be present** — Alice's wallet might be receiving shielded outputs before she upgrades her client. An INSERT-only would race with the daemon's observation writes; an UPSERT folds the two paths together: the daemon writes the row first if observation happens first, registration writes it first if registration happens first, and whichever runs second fills in the missing half. The daemon-vs-registration column ownership is disjoint — the daemon only writes `(address, bip32_account)`, the registration path owns `(wallet_id, index, ct_address, scan_privkey, catchup_state)` — so the UPSERT has no contended columns and no risk of either side overwriting the other's authoritative value.

## Why pre-derive scan privkeys instead of HD-derive on demand?

The Lambda environment makes per-request HD derivation a measurable cost; pre-derivation moves it to the gap-extension boundary which is rare. Storage cost of the per-index child privkeys is small.

## Why not store the spend privkey too?

The wallet-service must never hold the spend privkey. This is the entire point of the scan/spend split: the wallet-service can read but not steal. Spending is a client responsibility.

## Why a separate `shielded_status` instead of reusing `wallet.status`?

A wallet can be transparent-ready while shielded-catching-up. Conflating these into a single status enum forces the client to disambiguate via auxiliary fields. A separate column makes the lifecycle explicit.

## Why not let the client run the catch-up scan and POST results?

Rejected because it inverts the trust boundary — the wallet-service would believe the client about which historical outputs it owns, opening a balance-spoofing surface. With server-side catch-up, the wallet-service computes ownership against on-chain commitments it indexed itself.

## Why not register `scan_pubkey` (rather than `scan_privkey`) and let the client send rewind requests?

This was the very early "view-only delegation" sketch. It has two problems: (a) it doubles client-server round-trips for every push notification (server detects, client must rewind, server stores), losing the daemon's ability to compute balances synchronously; and (b) it is functionally equivalent to handing the scan privkey, since whoever holds the scan privkey can do everything the server would do. The RFC explicitly chose the "wallet-service holds scan privkey" model because it puts the read-access trade-off front-and-centre at registration.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Catch-up rate limit / backpressure.** A single very active wallet upgrading should not starve other wallets' catch-ups. Likely a token-bucket per wallet plus a global max-concurrent-catchup-jobs setting. Detail deferred to implementation.
2. **`network_byte` value(s) for the shielded address encoding.** Out of scope for the wallet-service design; must come from the on-chain address-format spec.
3. **Wallet deletion / shielded-disable flow.** If a user wants to "forget" their shielded keys, what is the effect on shielded-row ownership fields on `address`, `address_balance`, and history? Out of scope; the existing transparent wallet has no delete-key flow either.

# Future possibilities
[future-possibilities]: #future-possibilities

## Encryption-at-rest for scan-side secrets

This design stores `wallet.scan_xpriv` and the per-index `scan_privkey` on shielded rows of `address` (`bip32_account = 1`) in plaintext. The scope of that exposure is bounded by the existing DB access-control posture, but encryption-at-rest is the right next step. Two strategies were evaluated during the design phase and are recorded here so the work is not lost when the team picks one up.

The wallet-service holds two distinct secrets per shielded-enabled wallet:

- `wallet.scan_xpriv` — the HD privkey for the scan path. Used only at gap-extension time. Held by the API.
- `address.scan_privkey` (on `bip32_account = 1` rows) — the per-index child privkey. Used by the daemon on every match (hot path). Read by the daemon.

### Option A — Application-level AES-GCM with KMS-managed master key (recommended)

- A symmetric master key (256-bit) lives in AWS KMS (`Hathor/wallet-service/shielded-keys`).
- The application calls `kms:GenerateDataKey` once per process to obtain a data encryption key (DEK), wrapped by the master key. The wrapped DEK is cached in memory; the plaintext DEK is held only in process memory.
- For each secret to encrypt: `ciphertext = AES-256-GCM(DEK, plaintext, nonce_random_96bit)`. The on-disk format is `nonce(12B) || ciphertext || tag(16B)`.
- Decryption reverses the operation; the DEK is decrypted via `kms:Decrypt` if the in-memory DEK has been evicted.

**Pros.**

- Master key rotation is a KMS-side operation; the application can be ignorant.
- Decrypt audit logs run through CloudTrail.
- Engine-agnostic — the same code path works on MySQL or Postgres.
- Re-encryption (e.g., for compliance migration) is straightforward.

**Cons.**

- KMS API quota consumption; mitigated by caching the DEK.
- Requires `kms:GenerateDataKey` and `kms:Decrypt` IAM permissions on the daemon and the API task roles.
- The DEK is a runtime liability if a process is exploited; mitigated by short-lived processes (Lambda) and OS hardening (daemon).

### Option B — DB-native column-level encryption (MySQL `AES_ENCRYPT`)

- `AES_ENCRYPT(data, key, iv)` with the key supplied via session config at connection-acquisition time.

**Pros.**

- Zero application code beyond a SQL wrapper.
- Encryption boundary is the DB driver, not the application.

**Cons.**

- Key material is supplied **into** the DB session; if the DB connection log is compromised, the key may leak.
- Key rotation is a per-row re-encrypt operation that the application must drive.
- Auditability is weaker (no CloudTrail; only DB query logs, which themselves may contain key material).
- Several MySQL deployments disable `AES_ENCRYPT` for compliance reasons.

### Recommendation when this is picked up

Option A. The KMS-based pattern matches the rest of the AWS-native posture this repository already uses (alerts via SQS, deployments via Lambda). Audit-friendly and key-rotation-friendly.

The migration is a column-data change: read each row, encrypt the plaintext, write it back; flip the column-name (`scan_xpriv` → `scan_xpriv_ciphertext`) in a same-day deploy that ships the updated reader code.

The implementor would add an `encryption.ts` module with:

```ts
encryptScanPriv(plain: Buffer): Promise<Buffer>;
decryptScanPriv(cipher: Buffer): Promise<Buffer>;

encryptScanXpriv(plain: Buffer): Promise<Buffer>;
decryptScanXpriv(cipher: Buffer): Promise<Buffer>;
```

The two pairs use **different** encryption contexts (additional authenticated data) so a child privkey ciphertext cannot be substituted for an HD privkey ciphertext.

## Other future directions

- **Auditor / view-key delegation API.** A wallet could supply scan keys for an auditor account. Architecturally identical to the current upgrade flow with a different policy boundary.
- **Hardware-wallet-friendly upgrade.** The signature scheme on `scanXprivSignature` / `spendXpubSignature` could be exchanged for a more hardware-wallet-friendly one (e.g., a UR-encoded payload from an air-gapped device).
- **Optimistic catch-up.** When a wallet is mostly empty (typical for a fresh upgrade), the catch-up scan can short-circuit by checking against a Bloom filter of `address` values that have *ever* received a shielded output (derivable from `address` rows with `bip32_account = 1` and a non-zero `transactions` counter). Future optimisation.
- **Client-side scan key derivation verification.** A small endpoint could let the client verify the server derived the same spend-address set as it would, catching integration bugs at registration time.
