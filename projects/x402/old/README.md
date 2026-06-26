# Archived: the nano-contract x402 design (RFCs 0001–0004)

> **Status: superseded.** The RFCs in this folder describe the **first** design
> for x402 on Hathor, which routed every payment through an on-chain **nano
> contract escrow**. That approach was discarded. The current design — the
> `hathor-direct` scheme, which uses only regular UTXO transactions — lives in
> the parent folder (`projects/x402/0005`–`0008`). These documents are kept for
> provenance and for the trade-off analysis they contain; they are **not** the
> design we are building.

## What these documents proposed

| RFC | Title | Idea |
|---|---|---|
| [0001](0001-x402-support.md) | x402 Protocol Support for Hathor | Settle each x402 payment through an `X402Escrow` **nano contract**: the client `deposit()`s into a per-payment escrow contract; the facilitator `release()`s the funds to the seller (or the client reclaims them after a timelock). |
| [0002](0002-x402-client-sdk.md) | `@hathor/x402-client` | A `fetch` wrapper that, on a 402, creates the escrow nano contract, waits for confirmation, and retries with the proof. |
| [0003](0003-x402-server-middleware.md) | `@hathor/x402-server` | Express/Fastify/Hono middleware that emits 402, asks a facilitator to verify the escrow, and triggers `release()` settlement. |
| [0004](0004-x402-payment-channels.md) | x402 Payment Channels | An `X402Channel` nano contract to amortize the per-payment escrow cost: deposit once, then many off-path `spend()`s — explicitly introduced to work around 0001's per-payment latency. |

## Why it was discarded

**1. Time to execute a nano contract.** The escrow model needs the client's
`deposit()` transaction to be **confirmed by a block** before the seller will
release the resource — on the order of **~10 seconds per payment**. That breaks
the core x402 UX promise of a near-instant paid request, and it is the same
latency the wallet treats as "done" only after a broadcast. RFC 0004 (payment
channels) exists *solely* to hide this cost for high-frequency callers — i.e. the
base design needed a second, more complex contract just to become usable. A
design that needs a workaround to meet its own latency budget is the wrong
default.

**2. Not compatible with confidential transactions.** Hathor's planned
confidential transactions (Pedersen commitments + range proofs) apply to
**regular UTXO transactions**. Nano contracts need plaintext amounts to execute
blueprint logic, so a payment routed through an escrow contract can **never**
become confidential. Routing all x402 traffic through nano contracts would
forgo confidentiality for every x402 payment, indefinitely. (See
[`../0005-hathor-direct-scheme-research.md`](../0005-hathor-direct-scheme-research.md)
§3.9 for the analysis.)

**3. Operational weight.** An escrow/channel facilitator must hold a **funded,
seed-bearing wallet** to call `release()`/`spend()` (it pays fees and does PoW
for those on-chain actions). That is a hot key to protect, fund, and rotate, and
it makes the "public facilitator" hard to run. The `hathor-direct` facilitator
is **read-only** — no seed, no signing, no on-chain action — and can be run as a
stateless service.

## What replaced it

The `hathor-direct` scheme: the client broadcasts a **normal payment
transaction** and proves ownership of it by signing a server-issued challenge
(`requestId`) with the keys behind the transaction's inputs. The facilitator
only **reads** the full node to confirm the payment. One transaction per
payment, mempool-speed (~1–2 s), confidential-transaction-ready, and no
facilitator wallet.

- [`../0005-hathor-direct-scheme-research.md`](../0005-hathor-direct-scheme-research.md)
  — the investigation that compared the candidate UTXO schemes and recommended
  `hathor-direct`.
- [`../0006-x402-hathor-direct-protocol.md`](../0006-x402-hathor-direct-protocol.md)
  — the protocol specification.
- [`../0007-x402-roles-and-payment-flow.md`](../0007-x402-roles-and-payment-flow.md)
  — roles, end-to-end flow, and the facilitator API.
- [`../0008-x402-wallets-agents-and-skill.md`](../0008-x402-wallets-agents-and-skill.md)
  — how wallets, users, and agents interact with the protocol.

## Is the nano-contract escrow gone forever?

No. It remains the right tool for a **different** problem: high-value or
low-trust payments that need an **on-chain, protocol-enforced refund**.
`hathor-direct` deliberately trades that guarantee away for speed and
confidentiality (the client trusts the seller to deliver, exactly as with any
pre-paid API). If a protocol-level refund is ever required for a class of x402
payments, the escrow design in 0001 is the starting point — which is why these
documents are archived rather than deleted.
