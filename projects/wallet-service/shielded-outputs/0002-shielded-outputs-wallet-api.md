- Feature Name: `wallet_service_shielded_outputs_api`
- Start Date: 2026-05-04
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Extend the wallet-service REST API so client wallets can read and act on their shielded balances. **Balance** responses return the **complete balance** (transparent + shielded) in the existing `balance` and `transactions` fields — storage stays split per-kind, but the API merges before responding. **History** entries gain an optional `output_kind` discriminator and merged-by-time results from both the transparent and shielded history tables. **UTXO** endpoints take an optional `kind` query parameter (`transparent` | `shielded`); when omitted, the response is the merged set of both kinds. The wallet-service does not sign shielded spends — clients construct them from the per-output bytes returned. The change is on the same endpoint paths the API exposes today; no `/v2` namespace, no `Accept-Version` header, no migration plan.

# Motivation
[motivation]: #motivation

The [daemon and database design](0000-daemon-and-database.md) indexes shielded outputs into separate tables. The [wallet registration](0001-wallet-registration.md) populates the `shielded_address` lookup so ownership can be resolved. This document closes the loop: it specifies the read surfaces the wallet client uses to view its shielded balance, browse shielded history, and obtain the per-output data it needs to build a shielded spend.

The core API design tension is between **isolation** (clean separation of transparent and shielded reads) and **ergonomics** (a wallet UI typically wants one combined view per token, not two). The chosen resolution: keep the storage layer fully split (separate `tx_output` / `shielded_tx_output` tables, separate balance tables per kind), and **merge in the API**. The endpoint surfaces a single number per token, computed at read time from the two storage halves. If a future feature needs the per-kind split surfaced (e.g., a privacy-aware wallet UI that wants to render the breakdown), the API can grow a query parameter or a new field — but the default contract returns the user's spendable total.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A developer building a wallet client thinks about three reads:

1. **"How much HTR / token X does the user have?"** → balance endpoint. Gets back the user's complete balance per token (transparent + shielded merged into one number).
2. **"What just happened to my wallet?"** → history endpoint. Gets back a paginated stream of movements, each tagged transparent or shielded so a privacy-aware UI can render shielded receives differently.
3. **"Build me a transaction that sends 1.0 HTR"** → UTXO endpoint. By default the response includes both transparent and shielded UTXOs; the client picks the inputs it can sign for.

A wallet client that does not know about shielded gets a response shape that is **a strict superset of what it received before**: the existing fields carry the same meaning (now reflecting transparent + shielded where applicable), with optional new fields that old clients can safely ignore.

The design's invariants:

- **Storage stays split.** `tx_output` / `shielded_tx_output` and the per-kind balance/history tables are unchanged. The merge happens at API read time. If a future feature needs the per-kind split surfaced (e.g., a "show me only my shielded balance" view), the existing storage already supports it.
- **The API merges by default.** `GET wallet/balances` returns the user's complete balance in the existing `balance` field — no extra block to read. `GET wallet/utxos` returns both kinds unless the client explicitly filters with `kind=transparent` or `kind=shielded`.
- **History items are typed.** A history row has `output_kind: 'transparent' | 'shielded'` so the client can render a shielded-receive icon, a "private" tag, etc. The discriminator is additive — old clients ignore it.
- **Privacy of shielded data ends at the wallet-service / wallet-client boundary.** Once the client authenticates as the wallet owner, it sees recovered values and tokens for its own outputs in the clear (the wallet-service has them in the clear too — it ran the rewind). For *other* wallets' shielded outputs, the wallet-service knows nothing more than the on-chain bytes.

## Worked example: balance call

Client requests:

```http
GET /wallet/balances HTTP/1.1
Authorization: Bearer ...
```

Wallet identity comes from the bearer authorizer (`bearerAuthorizer` in the existing API), not from a path parameter. Every wallet-scoped read endpoint follows this pattern.

Response (when the wallet has 1000 transparent + 2500 shielded HTR; `shielded_status = 'ready'`):

```jsonc
{
  "success": true,
  "balances": [
    {
      "token":            { "id": "00", "name": "Hathor", "symbol": "HTR" },
      "transactions":     50,                                // 42 transparent + 8 shielded
      "balance":          { "unlocked": 3500, "locked": 0 }, // 1000 + 2500 merged
      "tokenAuthorities": { "unlocked": { /* … */ }, "locked": { /* … */ } }
    },
    {
      "token":            { "id": "abc…def", "name": "Token", "symbol": "TKN" },
      "transactions":     1,                                 // 0 + 1 shielded
      "balance":          { "unlocked": 10, "locked": 0 },   // 0 + 10
      "tokenAuthorities": { "unlocked": { /* … */ }, "locked": { /* … */ } }
    }
  ],
  "shielded_status": "ready"
}
```

The existing per-entry shape (`token`, `transactions`, `balance`, `tokenAuthorities`) is **unchanged in structure**. `balance.unlocked` is now the user's complete spendable HTR — the sum of transparent and shielded `unlocked_balance` rows. `transactions` is the sum of transparent and shielded transaction counts. Storage stays split (in `address_balance`, `wallet_balance`, `shielded_address_balance`, `shielded_wallet_balance`); the merge happens in the handler.

The top-level response gains an optional `shielded_status` field carrying the wallet's current shielded lifecycle phase. A wallet currently catching up after an upgrade reports `shielded_status: 'catching-up'`; the partial shielded balance is reflected in `balance.unlocked` as the daemon recovers it. A client may render a "still syncing" badge while `shielded_status != 'ready'` and `shielded_status != 'none'`.

For a transparent-only wallet, `balance.unlocked` is exactly the transparent balance (the shielded summands are zero) and `shielded_status` is `'none'`.

Old clients reading `balance.unlocked` and `transactions` see numbers that match the user's spendable funds. The numerical interpretation is the same as before; what changed is that the merged number now happens to also include shielded recoveries when present.

## Worked example: history call

```http
GET /wallet/history?token_id=00&skip=0&count=20 HTTP/1.1
Authorization: Bearer ...
```

The endpoint's existing query parameters are unchanged: `token_id` (single token), `skip`, `count`. The response gains an optional `output_kind` field per entry:

```jsonc
{
  "success": true,
  "history": [
    {
      "tx_id":       "deadbeef…",
      "timestamp":   1746384000,
      "token_id":    "00",
      "balance":     150,
      "voided":      false,
      "version":     1,
      "output_kind": "shielded"
    },
    {
      "tx_id":       "cafebabe…",
      "timestamp":   1746380000,
      "token_id":    "00",
      "balance":     -50,
      "voided":      false,
      "version":     1,
      "output_kind": "transparent"
    }
  ]
}
```

`output_kind` is optional. Old clients ignore it; new clients use it to render shielded receives differently (e.g., a "private" badge). The handler merges `wallet_tx_history` and `shielded_wallet_tx_history` rows for the requested token, sorts by `timestamp DESC`, and applies `skip`/`count` to the merged result.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `GET wallet/balances` — extended

### Response shape

Same wrapping (`{ "success": true, "balances": [...] }`) and same per-entry shape (`token`, `transactions`, `balance`, `tokenAuthorities`). What changes:

- `balance.unlocked` and `balance.locked` are now the **merged** values — sum of the corresponding columns from `wallet_balance` and `shielded_wallet_balance` for the requested token.
- `transactions` is the **merged** count — sum of the per-kind transaction counters. (Edge case: a single tx that affects both a transparent and a shielded UTXO of the same wallet contributes to both counters, so the merged number can in principle double-count one tx. This matches how the existing transparent counter behaves across multiple addresses of the same wallet, so it is consistent rather than wrong.)
- `tokenAuthorities` is unchanged in semantics. Authority bits live in `address_balance` only — shielded outputs cannot carry mint/melt authority — so this field always reflects the transparent half.
- The top-level response gains an optional `shielded_status` field: `'none' | 'catching-up' | 'ready' | 'error'`.

```jsonc
{
  "success": true,
  "balances": [
    {
      "token":            { /* TokenInfo, unchanged */ },
      "transactions":     50,                                    // transparent + shielded counts merged
      "balance":          { "unlocked": 3500, "locked": 0 },     // transparent + shielded amounts merged
      "tokenAuthorities": { "unlocked": …, "locked": … }         // transparent only (shielded has no authorities)
    }
  ],
  "shielded_status": "ready"
}
```

The wallet-service computes the total at read time. Storage stays split per kind; surfacing the split to clients is reserved for a future feature.

### Implementation

```ts
async function getBalances(walletId: string, tokenId?: string) {
  const [transparent, shielded] = await Promise.all([
    db.getWalletBalance(mysql, walletId, tokenId),               // existing helper
    db.getShieldedWalletBalance(mysql, walletId, tokenId),       // new helper, mirror shape
  ]);
  return mergeByTokenId(transparent, shielded);   // sums per-token, preserves tokenAuthorities from transparent
}
```

`mergeByTokenId` performs a full outer join in application code; every token that appears in either result becomes one entry, with summed amounts and counts.

If the wallet's `shielded_status != 'ready'`, partial shielded balances are already merged into the response as the daemon recovers them — the daemon updates `shielded_wallet_balance` row-by-row during catch-up. `shielded_status: 'catching-up'` tells the client the merged total is still settling.

## `GET wallet/history` — extended

### Request

Query parameters are unchanged from today: `token_id` (single, defaults to HTR), `skip`, `count`. No cursor pagination; the existing offset-based pagination is preserved.

### Response

The wrapping (`{ "success": true, "history": [...] }`) and the existing per-entry fields (`tx_id`, `timestamp`, `token_id`, `balance`, `voided`, `version`) are unchanged. Each entry gains one optional field:

```jsonc
{
  "tx_id":       "…",
  "timestamp":   1746384000,
  "token_id":    "00",
  "balance":     150,
  "voided":      false,
  "version":     1,
  "output_kind": "shielded"   // NEW. "transparent" | "shielded".
}
```

`output_kind` is optional and present on every row; old clients that don't read it see no change. New clients use it to render shielded entries with a privacy indicator.

### Implementation

The handler reads both `wallet_tx_history` and `shielded_wallet_tx_history` for the requested `(walletId, token_id)` and applies `skip` / `count` to the merged result. SQL outline:

```sql
SELECT *, 'transparent' AS output_kind FROM (
  SELECT wth.balance, wth.timestamp, wth.token_id, wth.tx_id, wth.voided, t.version
    FROM wallet_tx_history wth
    LEFT OUTER JOIN transaction t ON t.tx_id = wth.tx_id
   WHERE wth.wallet_id = :wallet AND wth.token_id = :token
) AS t
UNION ALL
SELECT *, 'shielded' AS output_kind FROM (
  SELECT swth.balance, swth.timestamp, swth.token_id, swth.tx_id, swth.voided, t.version
    FROM shielded_wallet_tx_history swth
    LEFT OUTER JOIN transaction t ON t.tx_id = swth.tx_id
   WHERE swth.wallet_id = :wallet AND swth.token_id = :token
) AS s
ORDER BY timestamp DESC
LIMIT :skip, :count
```

For very active wallets, the two-table merge holds up to mid-five-digit history depths; beyond that an indexed combined view or a denormalised history table can be added as a future optimisation.

## `GET wallet/utxos` and `GET wallet/tx_outputs` — extended

The two existing endpoints (`txOutputs.getFilteredUtxos` and `txOutputs.getFilteredTxOutputs`) take their existing filter set as query parameters (`tokenId`, `addresses`, `biggerThan`, `smallerThan`, etc., per `bodySchema` in `packages/wallet-service/src/api/txOutputs.ts`) and return transparent rows today.

Add an optional `kind` query parameter:

| `kind` value | Behaviour |
|---|---|
| omitted (default) | Return both transparent and shielded UTXOs, merged into one list. |
| `transparent` | Return only transparent UTXOs (existing transparent-only behaviour, opt-in). |
| `shielded` | Return only shielded UTXOs. |

```
GET /wallet/utxos?tokenId=00&biggerThan=99
Authorization: Bearer …
```

The default returns the user's complete UTXO set so a wallet client building a transaction sees everything it can spend in one call.

### Response shape — merged list

Every entry has a `kind` discriminator (`'transparent'` or `'shielded'`) so a client can dispatch on it. Common fields appear on both kinds; kind-specific fields appear only on the matching entries:

| Field | Transparent | Shielded |
|-------|-------------|----------|
| `kind` | `'transparent'` | `'shielded'` |
| `tx_id` | ✓ | ✓ |
| `index` | ✓ (output index) | ✓ (concatenated index, see below) |
| `token_id` | ✓ | ✓ |
| `value` | ✓ (visible amount) | ✓ (recovered amount, in Hathor cents) |
| `timelock`, `heightlock`, `locked` | ✓ | ✓ |
| `voided`, `spent_by` | ✓ | ✓ |
| `address` | ✓ (base58 transparent address) | ✓ (the long shielded address string, base58 of 71-byte payload) |
| `authorities` | ✓ (mint/melt bits) | always 0 — shielded outputs cannot carry authorities |
| `mode` | — | `'AMOUNT_SHIELDED'` \| `'FULLY_SHIELDED'` |
| `shielded_index` | — | ✓ (BIP32 child index in the wallet) |
| `spend_partial_address` | — | ✓ (`hash(spend_pubkey)`, hex 20 bytes) |
| `commitment`, `ephemeral_pubkey`, `range_proof`, `script` | — | ✓ |
| `token_data` | — | ✓ (only for `AMOUNT_SHIELDED`) |
| `asset_commitment`, `surjection_proof` | — | ✓ (only for `FULLY_SHIELDED`) |

Concrete example (one transparent + one `AMOUNT_SHIELDED` entry):

```jsonc
{
  "success": true,
  "utxos": [
    {
      "kind":         "transparent",
      "tx_id":        "abc…",
      "index":        2,
      "token_id":     "00",
      "address":      "WT4n…",         // existing base58 transparent address
      "value":        500,
      "authorities":  0,
      "timelock":     null,
      "heightlock":   null,
      "locked":       false
    },
    {
      "kind":                  "shielded",
      "tx_id":                 "def…",
      "index":                 5,                  // concatenated (transparent + shielded) index
      "shielded_index":        7,                  // BIP32 index for this wallet
      "token_id":              "00",
      "address":               "Hsh1…",            // long shielded base58 string
      "spend_partial_address": "0x…",
      "value":                 150,                // recovered amount, Hathor cents
      "authorities":           0,
      "mode":                  "AMOUNT_SHIELDED",
      "token_data":            1,
      "commitment":            "0x…",
      "ephemeral_pubkey":      "0x…",
      "range_proof":           "0x…",
      "script":                "0x…",
      "timelock":              null,
      "heightlock":            null,
      "locked":                false
    }
    // FULLY_SHIELDED entries additionally include "asset_commitment" and "surjection_proof".
  ]
}
```

`index` for a shielded entry is the **concatenated index** (position in `prev_tx.outputs ++ prev_tx.shielded_outputs`), so the client can build an input by emitting `{ prev_tx_id: tx_id, prev_index: index }` exactly as it does for transparent UTXOs. The wallet-service does not sign or finalise transactions.

### Filter semantics across kinds

Existing query parameters are honoured for both kinds where they make sense:

- `tokenId` — applied to both. For shielded, matches `shielded_tx_output.recovered_token_id`.
- `addresses[]` — applied to both. For transparent, matches `tx_output.address`. For shielded, matches `shielded_address.shielded_address` (the long base58 form). A request with an `addresses[]` filter that contains only transparent addresses naturally returns only transparent UTXOs from `kind=` (omitted) — the shielded half filters down to empty.
- `biggerThan`, `smallerThan`, `maxOutputs`, `skipSpent`, `ignoreLocked` — applied uniformly.
- `txId` + `index` — locates a single UTXO. The handler tries `tx_output` first, falls through to `shielded_tx_output` (same dereference logic the daemon uses for inputs).
- `authority` — only meaningful for transparent (shielded outputs cannot carry authority). When `authority > 0`, shielded entries are excluded automatically.

### Why the wallet-service does not return blinding factors

The blinding factor is deterministically re-derivable from `(scan_privkey, ephemeral_pubkey)`. The client has the scan privkey too (it owns the seed); it re-derives blinding factors locally when constructing the spend. Avoiding blinding-factor exposure on the wire reduces secret material in the API response.

## Mempool reflection

Mempool shielded UTXOs are reflected in balances and history the same way transparent mempool UTXOs are today. The `balance.unlocked` and `shielded.balance.unlocked` numbers include mempool deltas exactly as the existing transparent flow does — no shape change beyond what is described above.

## Real-time updates

WebSocket clients (`packages/wallet-service/src/ws/`) already receive `'new-tx'` events. The new `'new-shielded-tx'` event (introduced in the [daemon and database design](0000-daemon-and-database.md)) carries the shielded slice. Clients render both events as one logical "incoming tx" by correlating on `tx_id`.

## Backward compatibility

Same paths, same request shapes — no `/v2`, no `Accept-Version`. The semantic shifts that matter for old clients are bounded and analysed here.

A wallet-service deployment will, in practice, see two kinds of clients in production:

- **Old clients connecting to wallets that have no shielded keys** — overwhelmingly the common case during rollout. The wallet was registered via the old client and never received the upgrade flow. `shielded_wallet_balance` is empty, `shielded_tx_output` matched against this wallet returns no rows, every endpoint produces output identical to today.
- **Old clients connecting to wallets that *do* have shielded keys** — the rare downgrade scenario where a user upgraded their client (registering shielded keys), then later opens an old client. This is the only case where any new behaviour is observable to old clients.

Per-endpoint analysis:

| Surface | Old client, no shielded keys on wallet | Old client, wallet has shielded UTXOs |
|---------|-----------------------------------------|----------------------------------------|
| `GET wallet/balances` | `balance.unlocked` and `transactions` are exactly today's transparent values (the shielded summands are zero). Unchanged. | `balance.unlocked` reflects `transparent + shielded`. Old client sees a higher number than the UTXOs it can fetch and sign for. **Cosmetic discrepancy**, no fund loss. |
| `GET wallet/history` | New `output_kind` field is ignored; rows from `shielded_wallet_tx_history` are absent because none exist. Identical to today. | Old client receives merged history — additional rows from the shielded ledger appear interleaved by timestamp. The new `output_kind` field is ignored; otherwise the row shape (`tx_id`, `timestamp`, `token_id`, `balance`, `voided`, `version`) matches the existing one. The user sees more activity than they could see before — generally desirable, not a break. |
| `GET wallet/utxos` (default merged) | List contains only transparent entries (no shielded UTXOs exist). The new `kind` field on each entry is ignored. Identical to today. | List contains heterogeneous entries. Old clients that filter by `addresses[]` (the typical pattern) only get transparent matches — shielded entries have `address = <long shielded base58>` which doesn't match transparent addresses. Old clients that don't filter by addresses see shielded entries; if they try to spend one as a transparent UTXO, signing fails locally. **Cosmetic / signing-time error**, no fund loss. |
| `GET wallet/status` | Optional `shielded_status` field ignored. Unchanged. | Same. |
| `POST wallet/init` | Old clients send only the existing fields; the server runs the existing flow. Unchanged. | Same. (Not affected by the shielded-keys reconciliation since the request omits shielded fields.) |
| WebSocket `'new-tx'` | Unchanged. | Old client doesn't subscribe to `'new-shielded-tx'`. Misses shielded receives in real time, but the next balance / history poll catches them up. |
| Push notifications | Unchanged. | Same as WebSocket — old client doesn't receive shielded push events. |

**Summary of risk:** No fund loss in any scenario. The downgrade scenario produces cosmetic discrepancies (inflated `balance.unlocked` vs. the spendable transparent UTXOs) and one signing-time failure mode (old client picks a shielded UTXO from the merged list without realising it). Both are recoverable; both signal to the user that they should re-upgrade their client. The deployment plan should ship the shielded-aware client release before users start receiving shielded outputs in volume.

If the cosmetic discrepancy in the downgrade scenario is judged unacceptable, the alternative is to make the UTXO `kind` filter default to `transparent` instead of merged — see § *Rationale and alternatives*.

## Code organisation

- `packages/wallet-service/src/api/balances.ts` — extend the existing `get` handler to call the new `getShieldedWalletBalance` helper alongside `getWalletBalance` and merge.
- `packages/wallet-service/src/api/txhistory.ts` — extend the existing `get` handler to merge `wallet_tx_history` and `shielded_wallet_tx_history` and emit `output_kind` per row.
- `packages/wallet-service/src/api/txOutputs.ts` — accept the optional `kind` query parameter. With `kind=transparent`, run the existing transparent path. With `kind=shielded`, dispatch to the new shielded helpers. With `kind` omitted (the default), run both in parallel and concatenate the results, attaching the per-entry `kind` discriminator before returning.
- `packages/wallet-service/src/db/shielded.ts` — new helpers (`getShieldedWalletBalance`, `getShieldedWalletTxHistory`, `getFilteredShieldedUtxos`) that mirror the transparent helpers in `db/index.ts`.

No new handler files; no parallel `walletV2/` directory.

# Drawbacks
[drawbacks]: #drawbacks

- **Two-table merge in history.** The `wallet_tx_history` ↔ `shielded_wallet_tx_history` merge runs on every history page request. Mitigation: existing offset-based pagination is preserved (`skip` / `count`), so the cost is bounded by `count`; integration tests should exercise the boundary cases (empty shielded, empty transparent, both kinds in the same tx).
- **UTXO endpoint payload size.** Range proofs are large (~675 bytes each). With merged-by-default semantics, a wallet with hundreds of shielded UTXOs may receive a multi-megabyte response on a no-`kind` query. Mitigation: existing `maxOutputs` filter caps the response; a future optimisation could return only metadata by default and a separate endpoint for proof bytes if it becomes a problem in practice. Clients that don't need the heavy fields can pass `kind=transparent`.
- **Heterogeneous UTXO list.** The merged list mixes transparent and shielded entries with different field sets, distinguished by a `kind` discriminator. Mitigation: the table in § *Reference-level explanation* enumerates the per-kind fields; clients that don't dispatch on `kind` may stumble on shielded entries (see § *Backward compatibility*).
- **Cosmetic discrepancy for old clients in the downgrade scenario.** A user who upgraded then opens an old client sees `balance.unlocked` higher than the transparent UTXOs they can spend, and the merged UTXO list contains entries they can't sign for. No fund loss; recoverable by re-upgrading. Discussed in § *Backward compatibility*.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why store split, but merge in the API?

Storage is split because the two halves are physically very different (`tx_output` is small + flat; `shielded_tx_output` carries large cryptographic blobs) and because the daemon's reorg/void/recovery flows are simpler when the two paths don't share a hot table. That argument applies at the **storage layer**.

The argument doesn't carry to the API: a wallet UI almost always wants one number per token. Forcing every consumer to perform the per-kind merge themselves multiplies the merge code across the client ecosystem and creates a per-client bug surface where someone forgets to add the two halves. Doing the merge once in the wallet-service is cheaper, more consistent, and lets clients render `balance.unlocked` as the user's spendable amount with no extra logic.

If a future feature needs the per-kind split exposed (e.g., a privacy-aware UI rendering "1000 transparent / 2500 shielded / 3500 total"), the API can grow a query parameter (`?include=split`) or a new field on the response — adding without breaking. The current default expresses the common case.

## Why default UTXO results to merged rather than transparent-only?

Considered both. The trade-off:

- **Merged default (chosen).** New clients get the user's full UTXO set in one call without having to know about the `kind` parameter. Old clients on a shielded-keyed wallet receive heterogeneous entries and may stumble (see § *Backward compatibility* for the analysis); old clients on a non-shielded wallet are unaffected.
- **Transparent default (alternative).** Old clients are guaranteed unchanged behaviour. New clients must opt in via `kind=shielded` (to fetch the other half) or `kind=all` (to fetch both). More verbose for new clients; safer for old clients in the downgrade scenario.

The merged default is preferred because the downgrade scenario it puts at risk is rare (a user who explicitly upgraded then chose to open an old client) and the failure mode is recoverable (signing-time error or cosmetic balance discrepancy, no fund loss).

## Why a UTXO endpoint rather than a higher-level "build me a transaction" endpoint?

The wallet-service does not hold the spend privkey, so it cannot sign — only build an unsigned skeleton. Building a skeleton centrally would bind the API to the shielded-spend transaction format and pull the wallet-service into a role (constructing range proofs, surjection proofs, balancing input/output commitments) it has no other reason to play. Returning UTXOs and letting the client build the transaction keeps the contract narrow: the client library tracks the spend format, the wallet-service tracks the indexer format.

## Why include the full on-chain bytes in the UTXO response rather than recovered material only?

The client likely needs both — on-chain bytes for the input reference and proof construction, recovered metadata for the user-visible amount. Returning both is wasteful for clients that only need one half but avoids two endpoints. The savings from splitting are small (single-digit kilobytes per UTXO) compared to the operational simplification of one endpoint.

## Why no shielded "send" / "spend" endpoint at all?

Out of scope by design. Any future "construct unsigned shielded spend" endpoint is a fast-follow that depends on (a) the spend wire-format being settled, and (b) a shielded-spend client library being stable enough to coordinate with.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Range-proof transmission strategy.** Inline as hex, or fetch-on-demand via a separate endpoint? Inline is simpler; fetch-on-demand saves bandwidth for clients that don't construct spends in a given session.
2. **History deep-paginate performance** at large wallet sizes. May need a denormalised combined-history table; left as a future optimisation.

# Future possibilities
[future-possibilities]: #future-possibilities

- **A combined history materialised view.** If the two-table merge becomes a hot path, pre-compute a denormalised combined-history table maintained by the daemon.
- **Server-side spend skeleton construction.** Once the spend wire-format is settled and a stable client library exists, the wallet-service could expose a `POST wallet/spend-shielded` endpoint (wallet identity from the bearer authorizer, mirroring the existing `tx/proposal` shape) that returns an unsigned transaction skeleton. The wallet-service still does not sign.
- **Auditor-facing read endpoints.** With view-key delegation ([wallet registration](0001-wallet-registration.md) § Future possibilities), an auditor could read balances on a delegated read scope. The shape here trivially extends.
- **GraphQL surface.** The split-and-merge pattern is a natural fit for GraphQL field-selection. Out of scope here, but worth keeping in mind if the team adopts GraphQL elsewhere.
- **Push-notification payload extension** to include token symbol/name resolution server-side, sparing the client a follow-up call. Trivial follow-up.
- **Shielded mint/melt support.** Tracked in detail in design doc 1's *Future possibilities* (the upstream [shielded mint/melt RFC](https://github.com/HathorNetwork/rfcs/blob/feat/shielded-outputs/text/0000-shielded-outputs-mint-melt.md)). API impact when it lands is small: a wallet that receives newly-minted shielded tokens sees them through the existing balance and history endpoints via the existing recovery flow, with no shape change. The merged `balance.unlocked` and the merged UTXO list naturally include the new shielded value outputs. Authority bits in `tokenAuthorities` continue to reflect transparent only — the upstream RFC keeps authority outputs transparent.
