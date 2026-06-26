- Feature Name: x402_wallets_agents_and_skill
- Start Date: 2026-06-26
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: André Cardoso <andre@hathor.network>, Pedro Ferreira <pedro@hathor.network>
- Status: **Draft.** Companion to [`0006-x402-hathor-direct-protocol.md`](0006-x402-hathor-direct-protocol.md) and [`0007-x402-roles-and-payment-flow.md`](0007-x402-roles-and-payment-flow.md). This RFC covers the **client side**: the wallet capabilities `hathor-direct` requires, how humans and autonomous agents drive a payment, and the reference agent integration (the `x402-pay` skill).

# Summary
[summary]: #summary

x402 exists so that **software can pay software** over HTTP. This RFC specifies
what a Hathor wallet must be able to do to participate in `hathor-direct`, and
how the three kinds of payer — an embedded library client, a human at a wallet,
and an autonomous agent — turn a `402` into a paid response. It also documents
the reference agent integration: a Claude Code **skill** (`x402-pay`) that drives
the whole flow against a `hathor-wallet-headless` backend with no human clicking
beyond a yes/no.

The protocol itself is normative in
[0006](0006-x402-hathor-direct-protocol.md); the roles and server side in
[0007](0007-x402-roles-and-payment-flow.md). This RFC is about the payer.

# Motivation
[motivation]: #motivation

A browser dApp can demo `hathor-direct`, but the real target is the agentic case:
a script, backend, or AI agent making programmatic paid calls with no UI. For
that to work, two things must be true:

1. The payer's **wallet** must expose exactly two capabilities the scheme needs —
   send a transaction, and **sign a message with the key of a specific address**.
   The second is the input-ownership proof; without it the payer cannot satisfy
   verification.
2. There must be a repeatable **automation pattern**: detect `402`, pay, prove,
   retry, surface — safely, idempotently, and without leaking secrets.

This RFC pins down the capability requirement (including a `hathor-wallet-headless`
endpoint added specifically for x402) and the automation pattern, using the
`x402-pay` skill as the worked reference.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The two wallet capabilities x402 needs

| Capability | Used for | Why |
|---|---|---|
| **Send a payment transaction** | Pay `amount` of `asset` to `payTo`. | The on-chain payment. An ordinary transfer — no escrow, no contract. |
| **Sign a message with a given address's key** | Sign the `requestId` once per **unique input address** of the payment tx. | The input-ownership proof ([0006](0006-x402-hathor-direct-protocol.md#how-the-payment-binds-to-the-request-no-on-chain-marker-needed)) — proves the payer funded the payment. |

Everything else (change handling, UTXO selection, broadcast, sync) is ordinary
wallet behavior. The scheme deliberately needs nothing exotic — that is what lets
existing wallet stacks support it.

## Wallet backends

Two shapes of backend satisfy the capabilities:

- **Embedded library** (`@hathor/wallet-lib`): the payer runs the wallet
  in-process. Best for a single self-contained client (a CLI, a backend). It can
  sign a message by address index directly.
- **Headless service** (`hathor-wallet-headless`): the payer talks to a local
  wallet over HTTP. Best for agents and servers that want the wallet as a
  separate, reusable process. It exposes `send-tx`, transaction lookup, and
  message signing as HTTP routes.

The flow is identical; only the calls differ.

## The three payers

- **Embedded library client.** A program with a seed that wants a paid resource:
  start wallet → `402` → `sendTransaction` → sign `requestId` by address index →
  retry with `PAYMENT-SIGNATURE`. The whole protocol is ~50 lines around two
  wallet calls.
- **Human (wallet / dApp).** A person clicks "pay"; the wallet builds the
  transaction and signs the challenge. The human experiences a normal "approve
  payment" step; the proof assembly is invisible.
- **Autonomous agent.** An AI agent encounters a `402` while doing a task. It pays
  on the user's behalf, conversationally, through a headless wallet — confirming
  intent once, never surfacing a popup. This is what the `x402-pay` skill
  implements.

## The autonomous payment loop

```
fetch resource
  └─ 402? ── no ──▶ use the response
       │ yes
       ▼
   parse accepts[0]: scheme, network, amount, asset, payTo, extra.requestId
       ▼
   guard: wallet network == 402 network?      ── no ──▶ ABORT (never pay cross-network)
       ▼
   send-tx: amount of asset → payTo           ──▶ txId
       ▼
   read txId's UNIQUE input addresses
       ▼
   for each input address: sign-message(requestId) ──▶ {address, signature}
       ▼
   PAYMENT-SIGNATURE = base64({ txId, signatures, requestId })
       ▼
   refetch resource with the header
       ├─ 200 ──▶ surface data (+ for upto: charged/refund summary)
       └─ 402 ──▶ surface reason; one bounded retry at most
```

## The reference integration: the `x402-pay` skill

`x402-pay` is a Claude Code skill — the agent-side companion to a `hathor-direct`
resource server. When a session's HTTP call returns `402` (or the user asks to
fetch a paid URL), the skill takes over and runs the loop above against a
`hathor-wallet-headless` instance, starting one in Docker if the user has none.
Its design choices are the interesting part, because they are the safety rules any
autonomous payer should follow:

- **Confirm intent once.** On an auto-detected `402`, it asks before doing
  anything on-chain ("this returned 402 for X HTR to W…; pay it?").
- **Ask, don't probe.** It never scans Docker, compose files, or the working
  directory to *guess* which wallet to use, and never silently assumes
  `localhost:8000`. It asks for the headless URL and wallet id, validates the URL
  is really a headless (a JSON `/wallet/status`, not some other app on that port),
  and remembers the answers (but **never the seed**) for next time.
- **Refuse cross-network.** It compares the headless's network to the `402`'s
  `network` and aborts before any transaction if they differ.
- **Treat mainnet as dangerous.** It never auto-selects mainnet; opting in carries
  an explicit warning that real funds move.
- **Bounded retries.** One payment attempt per `402`; it surfaces the verifier's
  `reason` rather than looping.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Required wallet capability: per-address message signing

`hathor-direct` requires the payer to sign the `requestId` with the key of **each
unique input address** of the payment transaction. Embedded `@hathor/wallet-lib`
exposes this as `signMessageWithAddress(message, addressIndex, pin)`. For the
headless backend it requires:

> **`POST /wallet/sign-message`** — sign an arbitrary message with a wallet
> address. Added to `hathor-wallet-headless` specifically for x402. A headless
> without it cannot produce the proof; the payer MUST detect the `404` and report
> the version requirement rather than failing opaquely.

This is a concrete dependency this RFC places on the wallet stack: shipping
`hathor-direct` support means shipping `POST /wallet/sign-message`.

## Building the proof (headless backend)

After `send-tx` returns the payment `txId`:

1. **Read the transaction and extract unique input addresses.**
   `GET /wallet/transaction?id=<txId>` → take `inputs[].decoded.address`, dedupe.
   This set is exactly what verification step 6 will require signatures for.
2. **Sign `requestId` once per address.**
   `POST /wallet/sign-message { message: requestId, address }` →
   `{ signature, address, index }`. Accumulate `{ address, signature }` pairs.
3. **Assemble the payload** ([0006](0006-x402-hathor-direct-protocol.md#the-payment-signature-header-payment-proof)):
   `{ x402Version: 2, scheme, network, payload: { txId, signatures, requestId } }`,
   then base64 the JSON into the `PAYMENT-SIGNATURE` header.
4. **Ordering matters.** The **first** `signatures` entry's address is the
   canonical payer — the blocklist subject, the dedup record, and (for `upto`) the
   refund recipient. A payer that cares which address receives a refund puts it
   first; otherwise the wallet's natural ordering is fine.

For a typical single-input payment from a fresh address, `signatures` has length
one. Multi-input payments (the wallet pulled several UTXOs) produce one entry per
distinct input address — all of which must sign, or verification fails with
`missing_signature_for_input:<addr>`.

## Embedded library backend

The same proof with `@hathor/wallet-lib`: build and run a `sendTransaction`
(output `amount` → `payTo`, token `asset`, change to the payer's address), then
`signMessageWithAddress(requestId, addressIndex, pin)` for each input address's
index. No HTTP wallet needed — suitable for a self-contained CLI or backend payer.

## Skill operating model (reference)

The `x402-pay` skill encodes the loop and safety rules as ordered steps:

1. **Establish the backend.** Reuse a saved config if present; otherwise *ask* for
   the headless URL and wallet id (suggested defaults are offered, never assumed),
   validate it is a headless, and start the wallet (or a Docker container) if
   needed. Wait for `statusCode === 3` (READY).
2. **Network-match guard.** Compare headless network to the `402`'s `network`
   (stripping the `hathor:` prefix). Mismatch → abort, no transaction.
3. **Pay.** `POST /wallet/send-tx` with the integer `amount`, `asset`, `payTo`;
   capture `hash` as `txId`.
4. **Prove.** Extract unique input addresses; sign `requestId` per address;
   assemble `PAYMENT-SIGNATURE`.
5. **Retry + surface.** Refetch with the header. On `200`, return `data` (for
   `upto`, also summarize `chargedAmount` / `refundAmount` / `refundTxId`). On
   `402`, surface the `reason`.
6. **Remember (offer once).** Save `HEADLESS_URL`, `WALLET_ID`,
   `HEADLESS_NETWORK` for future sessions. **Never** the seed.

## Client error handling

| Verifier/Server reason | Meaning | Client action |
|---|---|---|
| `request_id_expired` | The `402`'s `requestId` is past its TTL. | Re-fetch the URL for a fresh `402`; **reuse the same on-chain tx**, re-sign the new `requestId`. The payment need not be remade. |
| `bad_signature:<addr>` | A signature didn't verify. | Re-sign for that address; if persistent, the wallet's signing is wrong/keys changed. |
| `missing_signature_for_input:<addr>` | An input address wasn't signed. | The unique-input set was incomplete; recompute it from the tx and sign all. |
| `replay` | This coin was already redeemed for a different request. | Do not retry; this payment is spent. (Normally only happens on a buggy reuse.) |
| `blocklisted` | The payer address is blocked (prior double-spend). | Stop; the address has lost standing. |
| `awaiting_block` | Amount above the zero-conf threshold; needs a confirming block. | Wait ~30 s and retry the same proof. |
| send-tx `insufficient funds` | Wallet underfunded. | Surface the payer address; fund it (testnet faucet) and retry. |

The `request_id_expired` row is the important property: a `hathor-direct` payment
is **decoupled** from any single challenge. The on-chain transaction is a fact;
the challenge is just what the payer is currently proving ownership against. An
expired challenge costs a re-sign, not a re-payment.

## Security requirements for an autonomous payer

These are normative for any unattended payer (the skill follows them):

- **Never persist the seed.** Take it once at wallet start; do not write it to
  memory, logs, or saved config.
- **Never pay cross-network.** Refuse if the wallet's network ≠ the `402`'s.
- **Never silently use mainnet.** Mainnet requires an explicit, warned opt-in.
- **Bound retries.** One payment per `402`; surface failures instead of looping
  (a loop can broadcast repeated real payments).
- **Keep the headless private.** `hathor-wallet-headless` has **no
  authentication** and signs real transactions; it MUST be reachable only by its
  payer process, never publicly exposed. (A publicly reachable headless in a demo
  is a convenience, not a pattern to copy.)

# Drawbacks
[drawbacks]: #drawbacks

- **New wallet endpoint dependency.** `POST /wallet/sign-message` must ship in the
  headless; older headless versions can pay but cannot prove, and fail at the
  signing step.
- **Per-input signing cost.** A multi-input payment needs one signature per unique
  input address. Wallets that consolidate many small UTXOs produce larger proofs;
  payers that want a one-signature proof should fund from a single UTXO.
- **Agent trust surface.** An autonomous payer moves real funds; the safety rules
  above are essential and easy to get wrong (silent network mismatch, unbounded
  retry, seed leakage).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- **Why per-address signing rather than a single wallet-level signature?** The
  proof must bind to the *funding sources* of the specific transaction. Signing
  with each input address's key is what proves the payer controls those inputs;
  a single arbitrary signature would not tie to the payment.
- **Why a headless backend for agents (vs. embedding the lib)?** A separate wallet
  process is reusable across calls and sessions, isolates key material from the
  agent's own process, and matches how agents already shell out to local tools.
  The embedded library remains the right choice for self-contained clients.
- **Why "ask, don't probe" in the skill?** Auto-discovering a wallet by scanning
  the environment risks paying from the wrong wallet or on the wrong network.
  Explicit confirmation is the safe default when real funds move.

# Prior art
[prior-art]: #prior-art

- **x402 client SDKs** (https://docs.x402.org/) — the `fetch`-wrapper pattern
  (intercept `402`, pay, retry) this RFC's loop follows.
- **Bitcoin "sign message"** — the per-address signing primitive reused for the
  ownership proof.
- The archived [`old/0002-x402-client-sdk.md`](old/0002-x402-client-sdk.md) — the
  earlier (escrow-based) client SDK design; the `fetch`-wrapper ergonomics carry
  over, the escrow internals do not.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Packaged client SDK.** Should the loop ship as `@hathor/x402-client` (a
  `fetch` wrapper over both backends), as the archived 0002 proposed, now adapted
  to `hathor-direct`?
- **`sign-message` release.** The exact `hathor-wallet-headless` version that
  includes `POST /wallet/sign-message`, to cite as the hard requirement.
- **Budgeting for agents.** Per-session/per-counterparty spend caps for unattended
  payers — protocol concern or purely client policy?
- **UTXO selection hint.** Should requirements optionally request single-UTXO
  funding to keep proofs one-signature?

# Future possibilities
[future-possibilities]: #future-possibilities

- **First-class wallet UX** for "approve x402 payment" in Hathor wallets, so human
  payers get a recognizable, auditable prompt.
- **MCP / tool-call exposure** so any agent framework (not just Claude Code) can
  call `pay-x402(url)` as a tool.
- **Spend policy + receipts.** Signed, stored `PAYMENT-RESPONSE` receipts and
  budget enforcement for unattended agents.
- **Confidential payments** are transparent to the payer: once CT ships, the same
  two wallet capabilities produce a confidential payment with no client change.
