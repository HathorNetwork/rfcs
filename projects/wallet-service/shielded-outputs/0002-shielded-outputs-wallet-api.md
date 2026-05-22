- Feature Name: `wallet_service_shielded_outputs_api`
- Start Date: 2026-05-12
- RFC PR: (to be filled)
- Hathor Issue: (to be filled)
- Author: André Carneiro

# Summary
[summary]: #summary

Extend the wallet-service REST API so client wallets can read and act on their shielded balances. The storage layer is unified: `tx_output` carries both transparent and shielded rows discriminated by `tx_output.mode`, while `address_balance` and `wallet_balance` carry parallel transparent and shielded amount columns on the same `(address, token_id)` / `(wallet_id, token_id)` row. This document specifies how that unified storage surfaces to clients.

**Balance** responses keep today's `balance.unlocked` / `balance.locked` shape as a merged total (back-compat), and expose a `?include=split` query parameter that returns `{ transparent, shielded, total }`. **History** entries gain an optional `output_kind` discriminator and an optional `balanceBreakdown` block, all read from a single `wallet_tx_history` row per `(wallet_id, tx_id, token_id)`. **UTXO** endpoints take an optional `kind` filter (`transparent` | `shielded`); without it the response includes both kinds (single-table query, no UNION). The wallet-service does not sign shielded spends — clients construct them from the per-output bytes returned. The change is on the same endpoint paths the API exposes today; no `/v2` namespace, no `Accept-Version` header.

# Motivation
[motivation]: #motivation

The [daemon and database design](0000-daemon-and-database.md) extends `tx_output`, `address`, `address_balance`, `wallet_balance`, `address_tx_history`, and `wallet_tx_history` to carry shielded data alongside transparent. The [wallet registration design](0001-wallet-registration.md) populates the shielded rows of the unified `address` table (`bip32_account = 1`) so ownership can be resolved. This document closes the loop: it specifies the read surfaces the wallet client uses to view its shielded balance, browse shielded history, and obtain the per-output data it needs to build a shielded spend.

The core API design tension is between **backward compatibility** (old clients reading `balance.unlocked` as a single number expect a meaningful number) and **breakdown** (a privacy-aware UI wants to render transparent vs shielded explicitly). The chosen resolution: keep the existing field shapes as merged totals by default, and expose the breakdown via opt-in query parameters and new optional response fields. Old clients see numbers that match the user's spendable funds; new clients can read the breakdown without an extra request shape.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A developer building a wallet client thinks about three reads:

1. **"How much HTR / token X does the user have?"** → balance endpoint. Gets back the user's complete balance per token (transparent + shielded merged). Pass `?include=split` to see the breakdown.
2. **"What just happened to my wallet?"** → history endpoint. Gets back a paginated stream of movements, each carrying an `output_kind` tag and an optional `balanceBreakdown` so a privacy-aware UI can render shielded receives differently.
3. **"Build me a transaction that sends 1.0 HTR"** → UTXO endpoint. By default the response includes both transparent and shielded UTXOs; the client picks the inputs it can sign for.

A wallet client that does not know about shielded gets a response shape that is **a strict superset of what it received before**: the existing fields carry the same meaning (now reflecting transparent + shielded where applicable), with optional new fields that old clients can safely ignore.

The design's invariants:

- **One row per `(wallet_id, tx_id, token_id)` in history.** A mixed-kind tx contributes to `balance_delta` (transparent) and `shielded_balance_delta` (shielded) on the **same row**. The API derives `output_kind: 'transparent' | 'shielded' | 'mixed'` from which deltas are non-zero.
- **One `tx_output` row per UTXO.** No UNION across two tables. The handler filters by `mode` when the client asks for `kind=transparent` / `kind=shielded`.
- **The default for new endpoints is to behave like the old endpoint.** Adding shielded support to a client should not require changing the request shape — only reading the new optional fields.
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
      "transactions":     50,                                 // combined count
      "balance":          { "unlocked": 3500, "locked": 0 },  // 1000 + 2500 merged
      "tokenAuthorities": { "unlocked": { /* … */ }, "locked": { /* … */ } }
    }
  ],
  "shielded_status": "ready"
}
```

With `?include=split`:

```jsonc
{
  "success": true,
  "balances": [
    {
      "token":            { "id": "00", "name": "Hathor", "symbol": "HTR" },
      "transactions":     50,
      "balance": {
        "unlocked": { "transparent": 1000, "shielded": 2500, "total": 3500 },
        "locked":   { "transparent": 0,    "shielded": 0,    "total": 0 }
      },
      "tokenAuthorities": { "unlocked": { /* … */ }, "locked": { /* … */ } }
    }
  ],
  "shielded_status": "ready"
}
```

The existing per-entry shape (`token`, `transactions`, `balance`, `tokenAuthorities`) is **unchanged at the field-name level**. `balance.unlocked` is a number in the default response (merged total) and an object with `{ transparent, shielded, total }` when `?include=split` is set. Storage is one row per `(wallet_id, token_id)` in `wallet_balance` carrying four amount columns; the handler picks which fields to render.

For a transparent-only wallet, `balance.unlocked` is exactly the transparent balance (the shielded summands are zero) and `shielded_status` is `'none'`.

Old clients reading `balance.unlocked` and `transactions` see numbers that match the user's spendable funds. The numerical interpretation is the same as before; what changed is that the merged number now happens to also include shielded recoveries when present.

## Worked example: history call

```http
GET /wallet/history?token_id=00&skip=0&count=20 HTTP/1.1
Authorization: Bearer ...
```

The endpoint's existing query parameters are unchanged: `token_id` (single token), `skip`, `count`. The response gains two optional fields per entry:

```jsonc
{
  "success": true,
  "history": [
    {
      "tx_id":       "deadbeef…",
      "timestamp":   1746816000,
      "token_id":    "00",
      "balance":     150,                  // = balance_delta + shielded_balance_delta
      "voided":      false,
      "version":     1,
      "output_kind": "shielded",           // NEW. "transparent" | "shielded" | "mixed"
      "balanceBreakdown": {                // NEW, optional
        "transparent": 0,
        "shielded":    150
      }
    },
    {
      "tx_id":       "cafebabe…",
      "timestamp":   1746812000,
      "token_id":    "00",
      "balance":     -50,
      "voided":      false,
      "version":     1,
      "output_kind": "transparent",
      "balanceBreakdown": {
        "transparent": -50,
        "shielded":    0
      }
    }
  ]
}
```

`output_kind` is `'transparent'` when only `balance_delta` is non-zero, `'shielded'` when only `shielded_balance_delta` is non-zero, and `'mixed'` when both are. `balance` is the sum (the old shape) so old clients keep working. `balanceBreakdown` carries the per-kind split for new clients.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `GET wallet/balances` — extended

### Request

Existing request shape unchanged. One new optional query parameter:

| Parameter | Values | Default |
|-----------|--------|---------|
| `include` | `split` | omitted (merged response) |

### Response shape — default (merged)

```jsonc
{
  "success": true,
  "balances": [
    {
      "token":            { /* TokenInfo, unchanged */ },
      "transactions":     50,                                    // combined count
      "balance":          { "unlocked": 3500, "locked": 0 },     // transparent + shielded summed
      "tokenAuthorities": { "unlocked": …, "locked": … }         // transparent only
    }
  ],
  "shielded_status": "ready"
}
```

- `balance.unlocked` / `balance.locked` are the sum of `wallet_balance.unlocked_balance + unlocked_shielded_balance` and `wallet_balance.locked_balance + locked_shielded_balance` respectively.
- `transactions` is the row's `transactions` count — a combined per-token counter (a single tx that touches both kinds of the same token increments by 1, not 2). This matches the natural reading of "transactions for this token in this wallet".
- `tokenAuthorities` is unchanged in semantics. Authority bits live on the transparent columns only — shielded outputs cannot carry mint/melt authority.
- The top-level response gains an optional `shielded_status` field: `'none' | 'catching-up' | 'ready' | 'error'`.

### Response shape — `?include=split`

`balance.unlocked` and `balance.locked` are objects:

```jsonc
{
  "balance": {
    "unlocked": { "transparent": 1000, "shielded": 2500, "total": 3500 },
    "locked":   { "transparent": 0,    "shielded": 0,    "total": 0 }
  }
}
```

`total = transparent + shielded` is computed server-side so the client doesn't have to re-add.

### Implementation

```ts
async function getBalances(walletId: string, tokenId?: string, includeSplit: boolean = false) {
  const rows = await db.getWalletBalance(mysql, walletId, tokenId);   // existing helper, returns the new 4-column shape
  return rows.map(row => ({
    token:            row.token,
    transactions:     row.transactions,
    balance:          includeSplit ? splitShape(row) : mergedShape(row),
    tokenAuthorities: row.tokenAuthorities,
  }));
}

function mergedShape(row) {
  return {
    unlocked: row.unlocked_balance + row.unlocked_shielded_balance,
    locked:   row.locked_balance + row.locked_shielded_balance,
  };
}

function splitShape(row) {
  return {
    unlocked: {
      transparent: row.unlocked_balance,
      shielded:    row.unlocked_shielded_balance,
      total:       row.unlocked_balance + row.unlocked_shielded_balance,
    },
    locked: {
      transparent: row.locked_balance,
      shielded:    row.locked_shielded_balance,
      total:       row.locked_balance + row.locked_shielded_balance,
    },
  };
}
```

If the wallet's `shielded_status != 'ready'`, partial shielded balances are already reflected in the response as the daemon recovers them — the daemon updates `wallet_balance.unlocked_shielded_balance` row-by-row during catch-up. `shielded_status: 'catching-up'` tells the client the merged total is still settling.

## `GET wallet/history` — extended

### Request

Query parameters are unchanged from today: `token_id` (single, defaults to HTR), `skip`, `count`.

### Response

The wrapping (`{ "success": true, "history": [...] }`) and the existing per-entry fields (`tx_id`, `timestamp`, `token_id`, `balance`, `voided`, `version`) are unchanged. Each entry gains two optional fields:

```jsonc
{
  "tx_id":       "…",
  "timestamp":   1746816000,
  "token_id":    "00",
  "balance":     150,                  // balance_delta + shielded_balance_delta, summed
  "voided":      false,
  "version":     1,
  "output_kind": "shielded",           // NEW. derived from which deltas are non-zero.
  "balanceBreakdown": {                // NEW, optional
    "transparent": 0,
    "shielded":    150
  }
}
```

### Implementation

A single SELECT against `wallet_tx_history`:

```sql
SELECT wth.tx_id, wth.timestamp, wth.token_id,
       wth.balance_delta, wth.shielded_balance_delta,
       wth.voided, t.version
  FROM wallet_tx_history wth
  LEFT OUTER JOIN transaction t ON t.tx_id = wth.tx_id
 WHERE wth.wallet_id = :wallet AND wth.token_id = :token
 ORDER BY wth.timestamp DESC
 LIMIT :skip, :count
```

No UNION — one row per `(wallet_id, tx_id, token_id)`. The handler computes `balance = balance_delta + shielded_balance_delta` and `output_kind` from which of the two delta columns are non-zero.

## `GET wallet/utxos` and `GET wallet/tx_outputs` — extended

The two existing endpoints (`txOutputs.getFilteredUtxos` and `txOutputs.getFilteredTxOutputs`) take their existing filter set as query parameters (`tokenId`, `addresses`, `biggerThan`, `smallerThan`, etc., per `bodySchema` in `packages/wallet-service/src/api/txOutputs.ts`).

Add an optional `kind` query parameter:

| `kind` value | Behaviour |
|---|---|
| omitted (default) | Return both transparent and shielded UTXOs (no filter on `mode`). |
| `transparent` | `WHERE mode = 0` |
| `shielded` | `WHERE mode IN (1, 2)` |

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
| `token_id` | ✓ | ✓ (NULL for unrecovered `mode = 2`) |
| `value` | ✓ (visible amount) | ✓ (recovered amount, in Hathor cents; NULL for unrecovered) |
| `timelock`, `heightlock`, `locked` | ✓ | ✓ |
| `voided`, `spent_by` | ✓ | ✓ |
| `address` | ✓ (base58 transparent address) | ✓ (base58 on-chain spend address) |
| `shielded_address` | — | ✓ (the long 71-byte user-facing base58 string from `address.shielded_address` on the row with `bip32_account = 1`) |
| `authorities` | ✓ (mint/melt bits) | always 0 — shielded outputs cannot carry authorities |
| `mode` | `0` | `1` (AMOUNT_SHIELDED) or `2` (FULLY_SHIELDED) |
| `recovery_state` | — | `'unowned' \| 'recovered' \| 'recovery_failed'` |
| `shielded_index` | — | ✓ (BIP32 child index in the wallet, from `address.index` on the row with `bip32_account = 1`) |
| `commitment`, `ephemeral_pubkey`, `range_proof`, `script` | — | ✓ (from satellite `shielded_tx_output_data`) |
| `token_data` | — | ✓ (only for `mode = 1`) |
| `asset_commitment`, `surjection_proof` | — | ✓ (only for `mode = 2`) |

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
      "address":      "WT4n…",
      "value":        500,
      "authorities":  0,
      "mode":         0,
      "timelock":     null,
      "heightlock":   null,
      "locked":       false
    },
    {
      "kind":             "shielded",
      "tx_id":            "def…",
      "index":            5,                  // concatenated index across outputs ++ shielded_outputs
      "shielded_index":   7,                  // BIP32 index for this wallet
      "token_id":         "00",
      "address":          "WXB8…",            // on-chain spend address (base58)
      "shielded_address": "Hsh1…",            // long 71-byte user-facing base58 string
      "value":            150,                // recovered amount, Hathor cents
      "authorities":      0,
      "mode":             1,
      "recovery_state":   "recovered",
      "token_data":       1,
      "commitment":       "0x…",
      "ephemeral_pubkey": "0x…",
      "range_proof":      "0x…",
      "script":           "0x…",
      "timelock":         null,
      "heightlock":       null,
      "locked":           false
    }
    // FULLY_SHIELDED entries additionally include "asset_commitment" and "surjection_proof".
  ]
}
```

`index` for a shielded entry is the **concatenated index** (position in `prev_tx.outputs ++ prev_tx.shielded_outputs`), so the client can build an input by emitting `{ prev_tx_id: tx_id, prev_index: index }` exactly as it does for transparent UTXOs. The wallet-service does not sign or finalise transactions.

### Filter semantics across kinds

Existing query parameters are honoured for both kinds where they make sense:

- `tokenId` — applied to both. Matches `tx_output.token_id` (which is populated at observe time for transparent and `mode = 1`, and after recovery for `mode = 2`).
- `addresses[]` — applied to both. Matches `tx_output.address` (the on-chain address column, which holds both transparent and shielded spend addresses).
- `biggerThan`, `smallerThan`, `maxOutputs`, `skipSpent`, `ignoreLocked` — applied uniformly.
- `txId` + `index` — locates a single UTXO via the unified PK `(tx_id, index)`.
- `authority` — only meaningful for transparent (shielded outputs cannot carry authority). When `authority > 0`, shielded entries are excluded automatically.

Unrecovered shielded rows (`recovery_state = 'unowned'`) are excluded by default from UTXO results because their `value` is unknown — the API can't return a spendable entry without an amount. A future query parameter (`?include_unrecovered=true`) could surface them for debugging, but it's not in v1.

### Implementation

```ts
async function getFilteredUtxos(walletId: string, filters: Filters, kind?: 'transparent' | 'shielded') {
  // Single query against tx_output. mode dispatch is one optional WHERE clause.
  const rows = await db.queryUtxos(mysql, walletId, filters, kind);

  // Hydrate shielded entries with crypto bytes and ownership metadata.
  const shieldedIds = rows.filter(r => r.mode !== 0).map(r => [r.tx_id, r.index]);
  const satellite = await db.getShieldedTxOutputDataByIds(mysql, shieldedIds);
  // Ownership rows live on the unified `address` table at bip32_account = 1.
  const ownership = await db.getShieldedAddressByAddresses(mysql, rows.map(r => r.address));

  return rows.map(r => formatUtxo(r, satellite, ownership));
}
```

The `formatUtxo` step picks the kind-appropriate fields from § *Response shape* above.

### Why the wallet-service does not return blinding factors

The blinding factor is deterministically re-derivable from `(scan_privkey, ephemeral_pubkey)`. The client has the scan privkey too (it owns the seed); it re-derives blinding factors locally when constructing the spend. Avoiding blinding-factor exposure on the wire reduces secret material in the API response.

## Mempool reflection

Mempool shielded UTXOs are reflected in balances and history the same way transparent mempool UTXOs are today. The `balance.unlocked` and `shielded.balance.unlocked` numbers include mempool deltas exactly as the existing transparent flow does — no shape change beyond what is described above.

## Real-time updates

WebSocket clients (`packages/wallet-service/src/ws/`) already receive `'new-tx'` events. This design keeps that single event and extends its payload with optional shielded fields (defined in `0000-daemon-and-database.md` § *Push notifications and WebSocket updates*). Clients render an incoming tx as one logical event regardless of kind mix.

## Backward compatibility

Same paths, same request shapes — no `/v2`, no `Accept-Version`. The semantic shifts that matter for old clients are bounded and analysed here.

A wallet-service deployment will, in practice, see two kinds of clients in production:

- **Old clients connecting to wallets that have no shielded keys** — overwhelmingly the common case during rollout. The wallet was registered via the old client and never received the upgrade flow. `wallet_balance.unlocked_shielded_balance` is zero, `wallet_tx_history.shielded_balance_delta` is zero on every row, every endpoint produces output identical to today.
- **Old clients connecting to wallets that *do* have shielded keys** — the rare downgrade scenario where a user upgraded their client (registering shielded keys), then later opens an old client. This is the only case where any new behaviour is observable to old clients.

Per-endpoint analysis:

| Surface | Old client, no shielded keys on wallet | Old client, wallet has shielded UTXOs |
|---------|-----------------------------------------|----------------------------------------|
| `GET wallet/balances` (no `?include`) | `balance.unlocked` and `transactions` are exactly today's transparent values (the shielded summands are zero). Unchanged. | `balance.unlocked` reflects `transparent + shielded`. Old client sees a higher number than the transparent UTXOs it can fetch and sign for. **Cosmetic discrepancy**, no fund loss. |
| `GET wallet/history` | New `output_kind` and `balanceBreakdown` fields are ignored; `shielded_balance_delta` is 0 on every row so `balance` matches today's `balance_delta`. Identical to today. | Old client receives a merged `balance` per row that includes shielded contributions. The user sees deltas they couldn't see before — generally desirable, not a break. |
| `GET wallet/utxos` (default merged) | List contains only transparent entries (no shielded UTXOs exist). The new `kind` and `mode` fields on each entry are ignored. Identical to today. | List contains heterogeneous entries. Old clients that filter by `addresses[]` only get matches against the addresses they know — shielded spend_addresses are different from transparent addresses, so they don't appear unless the client explicitly includes them. Old clients that don't filter by addresses see shielded entries; if they try to spend one as a transparent UTXO, signing fails locally. **Cosmetic / signing-time error**, no fund loss. |
| `GET wallet/status` | Optional `shielded_status` field ignored. Unchanged. | Same. |
| `POST wallet/init` | Old clients send only the existing fields; the server runs the existing flow. Unchanged. | Same. (Not affected by the shielded-keys reconciliation since the request omits shielded fields.) |
| WebSocket `'new-tx'` | Unchanged. | Payload gains optional shielded fields; old clients ignore them and miss the shielded slice in real-time. The next balance/history poll catches them up. |
| Push notifications | Unchanged. | Same as WebSocket — optional shielded fields ignored by old clients. |

**Summary of risk:** No fund loss in any scenario. The downgrade scenario produces cosmetic discrepancies (inflated `balance.unlocked` vs. the spendable transparent UTXOs) and one signing-time failure mode (old client picks a shielded UTXO from the merged list without realising it). Both are recoverable; both signal to the user that they should re-upgrade their client. The deployment plan should ship the shielded-aware client release before users start receiving shielded outputs in volume.

If the cosmetic discrepancy in the downgrade scenario is judged unacceptable, the alternative is to make the UTXO `kind` filter default to `transparent` instead of merged — see § *Rationale and alternatives*.

## Code organisation

- `packages/wallet-service/src/api/balances.ts` — extend the existing `get` handler to recognise `?include=split` and render the breakdown when requested. Reads one row per `(wallet_id, token_id)` from the modified `wallet_balance`.
- `packages/wallet-service/src/api/txhistory.ts` — extend the existing `get` handler to emit `output_kind` and `balanceBreakdown` per row. Single-table read against `wallet_tx_history`; no UNION.
- `packages/wallet-service/src/api/txOutputs.ts` — accept the optional `kind` query parameter; translate it to a `WHERE mode` filter. Hydrate shielded entries with satellite data and shielded-address metadata.
- `packages/wallet-service/src/db/index.ts` — `getWalletBalance` returns the new four-column shape. New helpers `getShieldedTxOutputDataByIds` and `getShieldedAddressByAddresses` hydrate shielded UTXO entries.

No new handler files; no parallel `walletV2/` directory.

# Drawbacks
[drawbacks]: #drawbacks

- **UTXO endpoint payload size.** Range proofs are large (~675 bytes each). With merged-by-default semantics, a wallet with hundreds of shielded UTXOs may receive a multi-megabyte response on a no-`kind` query. Mitigation: existing `maxOutputs` filter caps the response; a future optimisation could return only metadata by default and a separate endpoint for proof bytes if it becomes a problem in practice. Clients that don't need the heavy fields can pass `kind=transparent`.
- **Heterogeneous UTXO list.** The merged list mixes transparent and shielded entries with different field sets, distinguished by a `kind` discriminator. Mitigation: the table in § *Reference-level explanation* enumerates the per-kind fields; clients that don't dispatch on `kind` may stumble on shielded entries (see § *Backward compatibility*).
- **Cosmetic discrepancy for old clients in the downgrade scenario.** A user who upgraded then opens an old client sees `balance.unlocked` higher than the transparent UTXOs they can spend, and the merged UTXO list contains entries they can't sign for. No fund loss; recoverable by re-upgrading. Discussed in § *Backward compatibility*.
- **Two response shapes for `balance.unlocked`.** Default returns a number; `?include=split` returns an object. Clients that introspect the type at runtime must dispatch. Mitigation: the query parameter is explicit at the request site, so the client knows in advance which shape to expect.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why merge by default and split via query parameter?

Backward compatibility. Old clients read `balance.unlocked` as a number and the merged total is the natural meaning of "the user's spendable balance". A privacy-aware UI that wants the breakdown opts in with one query-parameter character.

The alternative — surfacing the split by default — would either change the type of `balance.unlocked` (from number to object) and break old clients hard, or add a sibling field (`balanceBreakdown` always present, alongside merged `balance`) that ships the breakdown on every response. The query-parameter approach is the minimal back-compat-preserving shape.

## Why default UTXO results to merged rather than transparent-only?

The trade-off:

- **Merged default (chosen).** New clients get the user's full UTXO set in one call without having to know about the `kind` parameter. Old clients on a shielded-keyed wallet receive heterogeneous entries and may stumble (see § *Backward compatibility*); old clients on a non-shielded wallet are unaffected.
- **Transparent default (alternative).** Old clients are guaranteed unchanged behaviour. New clients must opt in via `kind=shielded` (to fetch the other half) or `kind=all` (to fetch both). More verbose for new clients; safer for old clients in the downgrade scenario.

The merged default is preferred because the downgrade scenario it puts at risk is rare (a user who explicitly upgraded then chose to open an old client) and the failure mode is recoverable (signing-time error or cosmetic balance discrepancy, no fund loss).

## Why a UTXO endpoint rather than a higher-level "build me a transaction" endpoint?

The wallet-service does not hold the spend privkey, so it cannot sign — only build an unsigned skeleton. Building a skeleton centrally would bind the API to the shielded-spend transaction format and pull the wallet-service into a role (constructing range proofs, surjection proofs, balancing input/output commitments) it has no other reason to play. Returning UTXOs and letting the client build the transaction keeps the contract narrow: the client library tracks the spend format, the wallet-service tracks the indexer format.

## Why include the full on-chain bytes in the UTXO response rather than recovered material only?

The client likely needs both — on-chain bytes for the input reference and proof construction, recovered metadata for the user-visible amount. Returning both is wasteful for clients that only need one half but avoids two endpoints. The savings from splitting are small (single-digit kilobytes per UTXO) compared to the operational simplification of one endpoint.

## Why one row per `(wallet_id, tx_id, token_id)` in history instead of two by kind?

A user querying their history wants one entry per tx, not two. Splitting by `kind` in the PK would surface implementation detail (a mixed tx becomes two rows that the API has to remerge for display) without changing storage size meaningfully. The two-column layout (`balance_delta`, `shielded_balance_delta`) on a shared row lets the API render `output_kind` per row as `transparent | shielded | mixed` by reading which columns are non-zero.

## Why no shielded "send" / "spend" endpoint at all?

Out of scope by design. Any future "construct unsigned shielded spend" endpoint is a fast-follow that depends on (a) the spend wire-format being settled, and (b) a shielded-spend client library being stable enough to coordinate with.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Range-proof transmission strategy.** Inline as hex (current proposal), or fetch-on-demand via a separate endpoint? Inline is simpler; fetch-on-demand saves bandwidth for clients that don't construct spends in a given session.
2. **History deep-paginate performance** at large wallet sizes. May need an additional index on `(wallet_id, token_id, timestamp DESC)` if pagination of deep history becomes slow. Defer to benchmark.
3. **Whether to expose `recovery_state` on UTXO entries.** Useful for debugging; potentially leaks "this output is owned by my wallet but the rewind failed" detail to anything with bearer-token access. Keep for v1; revisit if it surfaces as a leakage concern.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Server-side spend skeleton construction.** Once the spend wire-format is settled and a stable client library exists, the wallet-service could expose a `POST wallet/spend-shielded` endpoint (wallet identity from the bearer authorizer, mirroring the existing `tx/proposal` shape) that returns an unsigned transaction skeleton. The wallet-service still does not sign.
- **Auditor-facing read endpoints.** With view-key delegation ([wallet registration](0001-wallet-registration.md) § Future possibilities), an auditor could read balances on a delegated read scope. The shape here trivially extends.
- **GraphQL surface.** The merged/split pattern is a natural fit for GraphQL field-selection. Out of scope here, but worth keeping in mind if the team adopts GraphQL elsewhere.
- **Push-notification payload extension** to include token symbol/name resolution server-side, sparing the client a follow-up call. Trivial follow-up.
- **Shielded mint/melt support.** Tracked in detail in the [daemon and database design](0000-daemon-and-database.md) § *Future possibilities*. API impact when it lands is small: a wallet that receives newly-minted shielded tokens sees them through the existing balance and history endpoints via the existing recovery flow, with no shape change. The merged `balance.unlocked` and the merged UTXO list naturally include the new shielded value outputs. Authority bits in `tokenAuthorities` continue to reflect transparent only.
