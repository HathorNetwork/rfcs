# x402 on Hathor

[x402](https://docs.x402.org/) reuses HTTP `402 Payment Required` to let software
pay software for an HTTP resource. This folder specifies how x402 works on Hathor.

## Current design: `hathor-direct`

The active design is the **`hathor-direct`** scheme: a paid request settles with a
**single ordinary UTXO transaction**, and the payer proves it with a signature —
no nano contract, no facilitator wallet.

1. The server answers an unpaid request with `402` and a stateless, signed
   challenge (`requestId`).
2. The client pays by broadcasting a normal transaction to the seller's address,
   then signs the `requestId` with the keys behind every input of that
   transaction.
3. A **read-only** verifier (the server itself, or a shared facilitator) confirms
   the payment by reading the full node.

This is fast (mempool speed, ~1–2 s), **confidential-transaction-ready** (the
payment is a plain transfer), and needs **no seed on the verification path**. The
trade-off is a deliberately lighter trust model (no protocol-level refund; the
client trusts the seller to deliver, as with any pre-paid API).

A second variant, **`hathor-direct-upto`**, authorizes a maximum and refunds the
unused remainder — for metered/usage-billed routes.

## Reading order

| Doc | What it covers |
|---|---|
| [`0005-hathor-direct-scheme-research.md`](0005-hathor-direct-scheme-research.md) | The investigation that compared UTXO-only schemes and **recommended** `hathor-direct`. Background / decision record. |
| [`0006-x402-hathor-direct-protocol.md`](0006-x402-hathor-direct-protocol.md) | **The protocol.** Schemes, wire format (`402` requirements, `requestId`, `PAYMENT-SIGNATURE`, `PAYMENT-RESPONSE`), verification algorithm, replay, double-spend handling, confidentiality. |
| [`0007-x402-roles-and-payment-flow.md`](0007-x402-roles-and-payment-flow.md) | Roles (resource server / verifier / facilitator / client), end-to-end flow, the facilitator HTTP API, and the operational model. |
| [`0008-x402-wallets-agents-and-skill.md`](0008-x402-wallets-agents-and-skill.md) | The payer side: wallet capabilities required, how humans and autonomous agents pay, and the `x402-pay` agent skill. |

Start at 0006 for the protocol; read 0005 first if you want the "why this and not
nano contracts."

## Archived: the nano-contract design

The original design escrowed every payment in a **nano contract**. It was
discarded — chiefly because executing a nano contract per payment was too slow
(~10 s confirmation) and because nano contracts cannot become confidential. Those
RFCs (0001–0004) are kept for provenance in [`old/`](old/README.md).

## Status

Draft. 0006–0008 promote the 0005 recommendation into a full specification, based
on the proof-of-concept implementation. Open questions are listed at the end of
each RFC.
