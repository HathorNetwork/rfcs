- Feature Name: `wallet_service_shielded_outputs_registration`
- Start Date: 2026-05-04
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Extend the wallet-service registration flow so a wallet can attach **scan and spend keys for shielded outputs** either at creation or later. New wallets register with `{ xpubkey, auth_xpubkey, scan_xpriv, spend_xpub }` in a single call; existing transparent-only wallets attach the shielded keys via a dedicated upgrade endpoint that triggers a historical re-scan. Shielded address derivation is BIP32 with a gap-limit policy (default 20, matching transparent), but driven by **two paths in lockstep** — `m/44'/280'/1'/0/{i}` (scan) and `m/44'/280'/2'/0/{i}` (spend) at the same `shielded_index`. The scan HD privkey and every per-index `scan_privkey` are stored **in plaintext**; encryption-at-rest is out of scope and described in § *Future possibilities*.

# Motivation
[motivation]: #motivation

The [daemon and database design](0000-daemon-and-database.md) establishes that the daemon ingests every shielded output observed on chain and matches owned ones via `spend_partial_address`. That match table (`shielded_address`) has to be populated by *something*. This document specifies that "something":

- The API surface a wallet uses to declare its shielded keys (at creation or upgrade).
- The on-server derivation that turns those keys into `shielded_address` rows.
- The historical re-scan that lets an upgrading wallet retroactively claim the shielded outputs it received before it was registered as shielded-enabled.

A second motivation is operational: the wallet-service runs a Lambda-deployed API where CPU per request is cost-sensitive. Doing BIP32 child derivation on every API call is significantly more expensive than reading a cached row. Pre-derivation at gap-limit-extension time is the only design that keeps the read path cheap.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The wallet-service has always known one kind of wallet: a wallet identified by an xpub, with a gap-limited list of derived addresses. After this RFC, a wallet still has exactly **one identity** (one `wallet` row), but it can now also carry a **shielded layer**: an HD scan privkey, an HD spend xpub, and a derivation index it shares between them.

A developer adding shielded support to a wallet client thinks in terms of one operation: **initialise (or re-initialise) the wallet.** Call the existing wallet-load endpoint with whatever keys the client has — transparent xpub always, optionally `scan_xpriv` and `spend_xpub`. The server reconciles its stored state with the submitted keys: a new wallet is created, a transparent-only wallet that just received its first shielded keys transitions to a catch-up phase, an already-shielded wallet whose keys match is a no-op. The client never has to introspect server state to decide which endpoint to call — it always calls one endpoint with everything it has.

Address gap-extension is **not** a client-driven operation. It happens automatically server-side, exactly as it does for transparent today: as the daemon processes shielded outputs and `last_used_shielded_index` advances, the daemon derives more `shielded_address` rows to maintain `shielded_max_gap` consecutive unused trailing addresses. The client never asks to extend the gap; it just asks for the next available shielded address (mirroring `GET /addresses/new` on the transparent side).

The contributor must hold three concepts:

- **Shielded index.** A single integer that drives **both** scan and spend derivations at the same position. There is no scenario in this design where `shielded_index` differs between scan and spend; they advance together.
- **Pre-derivation.** When the API generates `shielded_address` row N, it derives `scan_pubkey[N]`, `spend_pubkey[N]`, and `scan_privkey[N]` synchronously, and writes all three to the row. Once inserted, no further derivation is needed at read time.
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
  "timestamp":            1746384000,
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
5. Pre-derives 20 shielded addresses (indexes 0..19) and inserts them into `shielded_address` with their `scan_privkey`, `scan_pubkey`, and `spend_pubkey` columns populated.
6. Enqueues a catch-up job for `wallet_alice`.
7. Returns 200 with the wallet's status payload, including `shielded_status: 'catching-up'`. The client then polls the existing wallet-status endpoint until `shielded_status` reaches `'ready'`.

The catch-up job (running on the daemon, see § *Re-scan procedure*):

1. Reads the 20 `spend_partial_address` values just inserted for `wallet_alice`.
2. Issues `SELECT ... FROM shielded_tx_output WHERE spend_partial_address IN (...) AND voided = FALSE` paginated.
3. For each match, runs the same recovery logic as the live ingest path (rewind, validate, write `recovered_value` / `recovered_token_id`, apply to balance/history).
4. Emits a `'shielded-catchup-progress'` event per page so the wallet client can render progress.
5. On completion, transitions `shielded_status` to `'ready'`.

If the catch-up surfaces shielded receives at high indices, the catch-up algorithm itself extends the gap (see § *Re-scan procedure* — the algorithm is a fixpoint loop that repeats with newly-derived addresses until `shielded_max_gap` consecutive unused trailing addresses remain). Once Alice's wallet is `shielded_status = 'ready'`, ongoing gap-extension during normal ingest mirrors the transparent path: as the daemon processes a new shielded output that uses a high-index address, it derives more `shielded_address` rows to keep the gap stable.

### Variant: completely new wallet with shielded support

The client calls the same load endpoint with the full set of keys. The server finds no existing `wallet_alice`, runs the existing transparent creation path (sets `status = 'creating'`), and **in the same atomic step** sets `shielded_status = 'catching-up'`, derives the initial 20 shielded addresses, and enqueues both the transparent and shielded async initialisers. Both halves run in parallel. The client polls until both `status = 'ready'` and `shielded_status = 'ready'`.

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
  "timestamp":            1746384000,
  "firstAddress":         "...",

  // Optional. All-or-none — if any is provided, all must be.
  "scanXpriv":            "...",   // BIP32 HD privkey at m/44'/280'/1'
  "spendXpub":            "...",   // BIP32 HD xpub  at m/44'/280'/2'
  "scanXprivSignature":   "...",   // signature over `scanXpriv || timestamp` by authXpubkey/0
  "spendXpubSignature":   "..."    // signature over `spendXpub || timestamp` by authXpubkey/0
}
```

Validation of the shielded fields:

- All-or-none: if any of `scanXpriv`, `spendXpub`, `scanXprivSignature`, `spendXpubSignature` is present, all must be (`400 Bad Request` otherwise).
- Both signatures verified against `authXpubkey/0` using the existing `verifySignature` helper (the same primitive used for `xpubkeySignature` and `authXpubkeySignature`).
- Server confirms the spend xpub is a real BIP32 xpub (parses without error). No equivalent check on `scanXpriv` is necessary beyond signature validation.

#### Reconciliation table

The handler reconciles its stored state against the submitted body. The matrix is:

| Wallet exists? | `wallet.scan_xpriv` already stored? | Shielded fields in request? | Server behaviour |
|---|---|---|---|
| No | n/a | No | Existing transparent creation path. `shielded_status = 'none'`. |
| No | n/a | Yes | Transparent creation path **and** shielded init in the same atomic step: set `scan_xpriv`, `spend_xpub`, `shielded_max_gap = 20`; pre-derive 20 shielded addresses; `shielded_status = 'catching-up'`; enqueue catch-up job. |
| Yes | No | No | Existing transparent already-loaded behaviour: return current status. |
| Yes | No | Yes | **Upgrade case.** Verify shielded signatures; set `scan_xpriv`, `spend_xpub`, `shielded_max_gap = 20`; pre-derive 20 shielded addresses; `shielded_status = 'catching-up'`; enqueue catch-up job; return current status. |
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

Concretely: when the daemon's shielded ingest flow recovers an output at `shielded_index = N`, it updates `wallet.last_used_shielded_index = max(N, current)` and, if `last_used_shielded_index + shielded_max_gap > highest_derived_shielded_index`, derives additional `shielded_address` rows in the same DB transaction. The reused helper is the `generateAddresses` library function (already used for transparent), parametrised with the spend xpub and scan xpriv.

The client requests "the next available shielded address" via the same endpoint that exists for transparent (`GET wallet/addresses/new`), extended to take an optional `kind=shielded` query parameter. That endpoint never derives — it only reads from the pool the daemon maintains.

## Address derivation

For each new shielded index `i`:

```ts
const scanXpriv  = wallet.scan_xpriv;   // plaintext
const spendXpub  = wallet.spend_xpub;

const scanChild  = bip32.derive(scanXpriv, `0/${i}`);   // child of m/44'/280'/1'
const spendChild = bip32.derive(spendXpub,  `0/${i}`);  // child of m/44'/280'/2'

const scanPriv   = scanChild.privateKey;     // 32 bytes
const scanPub    = scanChild.publicKey;      // 33 bytes compressed
const spendPub   = spendChild.publicKey;     // 33 bytes compressed

const spendPartialAddress = hash160(spendPub);            // 20 bytes — match key
const shieldedAddress     = encodeShieldedAddress(scanPub, spendPub);  // base58, ≤100 chars

await db.insertShieldedAddress({
  spend_partial_address: spendPartialAddress,
  wallet_id:             wallet.id,
  shielded_index:        i,
  shielded_address:      shieldedAddress,
  scan_privkey:          scanPriv,
  scan_pubkey:           scanPub,
  spend_pubkey:          spendPub,
});
```

`encodeShieldedAddress` produces the user-facing string:

```
base58( network_byte(1B) || scan_pubkey(33B) || spend_pubkey(33B) || checksum(4B) )
```

Where `checksum = first_4_bytes(SHA256(SHA256(network_byte || scan_pubkey || spend_pubkey)))`, mirroring the transparent address checksum pattern.

The `network_byte` distinguishes shielded mainnet, shielded testnet, and any future network discriminators. Concrete byte values are out of scope here; they belong with the on-chain address-format specification.

## Re-scan procedure

The catch-up job runs on the daemon (not in the Lambda API) because it is unbounded in duration and must not block API request capacity. The single trigger is the wallet-load endpoint reconciling a wallet that just received its first shielded keys (the upgrade case, or the new-shielded-wallet case). Catch-up starts with the freshly-derived initial gap of `shielded_max_gap` addresses (default 20).

Catch-up must respect the BIP32 gap-limit invariant: it cannot stop until there are `shielded_max_gap` consecutive unused trailing shielded addresses. If the initial gap surfaces high-index receives, the algorithm derives more addresses and continues. This mirrors the transparent restoration loop in `generateAddresses` (`do { … } while (lastUsedAddressIndex + maxGap > highestCheckedIndex)`).

Job algorithm (idempotent, fixpoint loop):

```
loop:
  1. Read the candidate spend_partial_address set:
       SELECT spend_partial_address, shielded_index, scan_privkey
       FROM shielded_address
       WHERE wallet_id = :wallet_id
         AND catchup_state = 'pending'

     If empty → goto step 4.

  2. Process each (spend_partial_address, scan_privkey) in batches of M:
       a. SELECT tx_id, output_index, mode, ephemeral_pubkey, commitment,
                 range_proof, asset_commitment, token_data, voided, first_block
          FROM shielded_tx_output
          WHERE spend_partial_address IN (...)
            AND voided = FALSE
            AND recovery_state = 'unowned'
          ORDER BY first_block ASC, tx_id ASC
          LIMIT M

       b. For each row:
            - rewindAmount/FullShieldedOutput
            - on success: UPDATE shielded_tx_output SET recovery_state='recovered',
                          recovered_value=..., recovered_token_id=...
                          WHERE tx_id=... AND output_index=...
                          AND recovery_state='unowned'
            - apply to shielded_address_balance / shielded_wallet_balance / history
            - if this row matched a higher-index address than the wallet's current
              last_used_shielded_index, advance last_used_shielded_index.

       c. Mark progress: UPDATE shielded_address SET catchup_state='done'
          WHERE spend_partial_address IN (...)

  3. Gap-extension check (the fixpoint step):
       last_used   = wallet.last_used_shielded_index   (-1 if no address has received yet)
       highest     = max(shielded_address.shielded_index) for this wallet
       max_gap     = wallet.shielded_max_gap

       If last_used + max_gap > highest:
         - Derive shielded_address rows for indexes (highest+1)..(last_used+max_gap),
           inserting them with catchup_state = 'pending'.
         - Update wallet.last_used_shielded_index if last_used was advanced in step 2.
         - goto step 1 — the loop will scan the freshly-derived addresses.

       Else (gap is satisfied):
         - goto step 4.

  4. Catch-up complete:
       UPDATE wallet SET shielded_status='ready' WHERE id=:wallet_id

  5. Emit a final 'shielded-catchup-complete' event.
```

Termination is guaranteed: each loop iteration either marks at least one address as `done` (decrementing the pending set) or reaches the gap-satisfied condition. The chain has finite history, so the highest used `shielded_index` for any wallet is bounded.

The `WHERE recovery_state = 'unowned'` filter on the row update makes the recovery write idempotent: a retry of the same output is a no-op.

The balance update is **not** idempotent on its own — the algorithm relies on the `recovery_state` transition guard to ensure a balance delta is applied at most once per `(tx_id, output_index)`.

Pagination size `M` is configurable; a reasonable default is 200 (~200 ms per batch at ~1 ms per rewind on commodity hardware).

`catchup_state` is a column on `shielded_address` (defined in the [daemon and database design](0000-daemon-and-database.md) § *Schema additions*); it lets the job resume cleanly across restarts. Newly-derived rows from the gap-extension step are inserted with `catchup_state = 'pending'`, so a job that crashes between iterations resumes cleanly on the next run.

### Re-scan ordering and reorgs

The catch-up job reads `shielded_tx_output` rows that the live ingest path is also writing to. Two interaction concerns:

- **Live writes during catch-up.** A new vertex containing a shielded output for a catching-up wallet is processed by the live path in the normal way (it sees the wallet's `shielded_address` row and recovers in-line). The catch-up job will then also see that row but the `recovery_state = 'unowned'` filter excludes already-recovered outputs. No double credit.
- **Reorgs during catch-up.** A reorg that voids a tx the catch-up job already credited reverses the balance through the standard `voidShieldedOutputs` path. The catch-up job does not need special reorg handling.

## Storage of scan-side secrets

Scan-side secrets are stored in **plaintext**:

- `wallet.scan_xpriv` — the HD privkey for the scan path. Used by the API at gap-extension time to derive new children.
- `shielded_address.scan_privkey` — the per-index child privkey. Used by the daemon on every match (hot path).

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
- **Plaintext scan secrets in the DB.** Anyone with read access to the `wallet` and `shielded_address` tables sees scan-side material in the clear, and can replay rewinds against any shielded output to learn amounts and tokens for any wallet whose secrets they have. Mitigation: the existing DB access-control posture (network isolation, IAM-scoped credentials) is the operative control. Encryption-at-rest is described in § *Future possibilities* as the next step.
- **More secret material in the DB.** Per-index `scan_privkey` is a row count proportional to `wallets × gap_limit`. At 10k wallets × 20 = 200k rows, each ~32 bytes. Negligible in storage, but more secret-material rows to govern. Mitigation: same DB access-control as above.
- **Two near-parallel signature-check paths** (transparent registration vs. shielded upgrade). Mitigation: share the `validateSignatures` helper.
- **Status surface for clients widens.** The shielded `catching-up` state is a new lifecycle phase clients must render. Mitigation: documented in the API spec; default rendering is "available, syncing".

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why pre-derive scan privkeys instead of HD-derive on demand?

Discussed in the [daemon and database design](0000-daemon-and-database.md) § *Why per-index pre-derived scan privkeys instead of HD-derive on demand?*. The Lambda environment makes per-request HD derivation a measurable cost; pre-derivation moves it to the gap-extension boundary which is rare. Storage cost of the per-index child privkeys is small.

## Why not store the spend privkey too?

The wallet-service must never hold the spend privkey. This is the entire point of the scan/spend split (option B from the upstream RFC's address-format choice): the wallet-service can read but not steal. Spending is a client responsibility.

## Why a separate `shielded_status` instead of reusing `wallet.status`?

A wallet can be transparent-ready while shielded-catching-up. Conflating these into a single status enum forces the client to disambiguate via auxiliary fields. A separate column makes the lifecycle explicit.

## Why not let the client run the catch-up scan and POST results?

Considered (Q5 option C). Rejected because it inverts the trust boundary — the wallet-service would believe the client about which historical outputs it owns, opening a balance-spoofing surface. With server-side catch-up, the wallet-service computes ownership against on-chain commitments it indexed itself.

## Why not register `scan_pubkey` (rather than `scan_privkey`) and let the client send rewind requests?

This was the very early "view-only delegation" sketch. It has two problems: (a) it doubles client-server round-trips for every push notification (server detects, client must rewind, server stores), losing the daemon's ability to compute balances synchronously; and (b) it is functionally equivalent to handing the scan privkey, since whoever holds the scan privkey can do everything the server would do. The RFC explicitly chose the "wallet-service holds scan privkey" model because it puts the read-access trade-off front-and-centre at registration.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Catch-up rate limit / backpressure.** A single very active wallet upgrading should not starve other wallets' catch-ups. Likely a token-bucket per wallet plus a global max-concurrent-catchup-jobs setting. Detail deferred to implementation.
2. **`network_byte` value(s) for the shielded address encoding.** Out of scope for the wallet-service design; must come from the on-chain address-format spec.
3. **Wallet deletion / shielded-disable flow.** If a user wants to "forget" their shielded keys, what is the effect on `shielded_address`, `shielded_address_balance`, and history? Out of scope; the existing transparent wallet has no delete-key flow either.

# Future possibilities
[future-possibilities]: #future-possibilities

## Encryption-at-rest for scan-side secrets

This design stores `wallet.scan_xpriv` and `shielded_address.scan_privkey` in plaintext. The scope of that exposure is bounded by the existing DB access-control posture, but encryption-at-rest is the right next step. Two strategies were evaluated during the design phase and are recorded here so the work is not lost when the team picks one up.

The wallet-service holds two distinct secrets per shielded-enabled wallet:

- `wallet.scan_xpriv` — the HD privkey for the scan path. Used only at gap-extension time. Held by the API.
- `shielded_address.scan_privkey` — the per-index child privkey. Used by the daemon on every match (hot path). Read by the daemon.

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
- **Optimistic catch-up.** When a wallet is mostly empty (typical for a fresh upgrade), the catch-up scan can short-circuit by checking against a Bloom filter of `spend_partial_address` values that have *ever* received a shielded output. Future optimisation.
- **Client-side scan key derivation verification.** A small endpoint could let the client verify the server derived the same `spend_partial_address` set as it would, catching integration bugs at registration time.
