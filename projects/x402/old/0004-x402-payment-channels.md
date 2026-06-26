- Feature Name: x402_payment_channels
- Start Date: 2026-04-02
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Andre Cardoso <andre@hathor.network>

# x402 Payment Channels for Hathor Network

---

## 1. Overview

This document describes **payment channels** for x402 on Hathor — a pre-funded nano contract that allows a client to make multiple x402 payments without creating a new escrow per request.

### 1.1 The Problem

The base x402 implementation (RFC 0001) creates **one X402Escrow contract per payment**. This works and provides strong guarantees, but has a cost:

- **~10 seconds** per payment (waiting for escrow deposit confirmation)
- **One on-chain transaction per request** from the client
- For high-frequency use (agent making 100 API calls), this means 100 contracts and ~16 minutes of cumulative wait time

### 1.2 The Solution: Pre-Funded Channels

A payment channel lets the client deposit funds once, then make many payments against that balance without additional on-chain transactions from the client. The facilitator calls `spend()` on the channel for each request — this is still on-chain, but the **client doesn't wait** because the facilitator handles it asynchronously after serving the resource.

```
Without channels (current):
  Request 1: client deposits → wait 10s → get data → facilitator releases
  Request 2: client deposits → wait 10s → get data → facilitator releases
  Request 3: client deposits → wait 10s → get data → facilitator releases
  Total: 30s of client waiting

With channels:
  Setup: client deposits 10 HTR into channel → wait 10s (once)
  Request 1: client sends channelId → facilitator verifies balance → get data → facilitator spends
  Request 2: client sends channelId → facilitator verifies balance → get data → facilitator spends
  Request 3: client sends channelId → facilitator verifies balance → get data → facilitator spends
  Total: 10s of client waiting (setup only)
```

After the initial channel deposit, each subsequent request is **instant** from the client's perspective — no wallet interaction, no on-chain wait.

---

## 2. X402Channel Blueprint

### 2.1 Design

```python
from hathor import (
    Address, Amount, Blueprint, Context, NCDepositAction, NCFail,
    NCWithdrawalAction, Timestamp, TokenUid, export, public, view
)

PHASE_OPEN = 'OPEN'
PHASE_CLOSED = 'CLOSED'


@export
class X402Channel(Blueprint):
    buyer: Address
    facilitator: Address
    token_uid: TokenUid
    total_deposited: Amount
    total_spent: Amount
    phase: str
    deadline: Timestamp

    @public(allow_deposit=True)
    def initialize(
        self,
        ctx: Context,
        facilitator: Address,
        token_uid: TokenUid,
        deadline: Timestamp,
    ) -> None:
        action = ctx.get_single_action(token_uid)
        if not isinstance(action, NCDepositAction):
            raise NCFail("Must include a deposit action")

        self.buyer = ctx.get_caller_address()
        self.facilitator = facilitator
        self.token_uid = token_uid
        self.total_deposited = action.amount
        self.total_spent = 0
        self.phase = PHASE_OPEN
        self.deadline = deadline

    @public(allow_withdrawal=True)
    def spend(self, ctx: Context, amount: Amount, seller: Address) -> None:
        caller = ctx.get_caller_address()
        if caller != self.facilitator:
            raise NCFail("Only the facilitator can spend")
        if self.phase != PHASE_OPEN:
            raise NCFail("Channel is not open")
        if self.total_spent + amount > self.total_deposited:
            raise NCFail("Insufficient channel balance")

        action = ctx.get_single_action(self.token_uid)
        if not isinstance(action, NCWithdrawalAction):
            raise NCFail("Must include a withdrawal action")
        if action.amount != amount:
            raise NCFail("Withdrawal amount must match spend amount")

        self.total_spent = self.total_spent + amount

    @public(allow_deposit=True)
    def top_up(self, ctx: Context) -> None:
        caller = ctx.get_caller_address()
        if caller != self.buyer:
            raise NCFail("Only the buyer can top up")
        if self.phase != PHASE_OPEN:
            raise NCFail("Channel is not open")

        action = ctx.get_single_action(self.token_uid)
        if not isinstance(action, NCDepositAction):
            raise NCFail("Must include a deposit action")

        self.total_deposited = self.total_deposited + action.amount

    @public(allow_withdrawal=True)
    def close(self, ctx: Context) -> None:
        caller = ctx.get_caller_address()

        if self.phase != PHASE_OPEN:
            raise NCFail("Channel is already closed")

        if caller != self.buyer and caller != self.facilitator:
            if ctx.block.timestamp < self.deadline:
                raise NCFail("Only buyer or facilitator can close before deadline")

        remaining = self.total_deposited - self.total_spent
        if remaining > 0:
            action = ctx.get_single_action(self.token_uid)
            if not isinstance(action, NCWithdrawalAction):
                raise NCFail("Must include a withdrawal action")
            if action.amount != remaining:
                raise NCFail("Must withdraw exact remaining balance")

        self.phase = PHASE_CLOSED

    @view
    def get_remaining(self) -> Amount:
        return self.total_deposited - self.total_spent
```

### 2.2 Key Differences from X402Escrow

| | X402Escrow | X402Channel |
|---|---|---|
| **Contracts per payment** | 1 per request | 1 per session (many requests) |
| **Client on-chain txs** | 1 per request | 1 to open + optional top-ups |
| **Client wait time** | ~10s per request | ~10s once (at channel open) |
| **Facilitator action** | `release()` — all-or-nothing | `spend(amount, seller)` — incremental |
| **Seller specified** | At escrow creation | At each spend |
| **Multiple sellers** | No (one per escrow) | Yes (different seller per spend) |
| **Refund** | Full amount back to buyer | Remaining balance back to buyer |
| **Top-up** | Not possible | Buyer can add more funds |

### 2.3 Channel Lifecycle

```
                    ┌────────────────────────────────┐
                    │            OPEN                 │
  initialize()      │                                │
  + deposit ───────▶│  Balance: total_deposited      │◀──── top_up()
                    │  Spent:   total_spent           │      + deposit
                    │  Remaining: deposited - spent   │
                    │                                │
                    │  spend() can be called          │
                    │  many times by facilitator      │
                    │                                │
                    └───────────────┬────────────────┘
                                    │
                       close()      │
                       buyer/facilitator anytime
                       anyone after deadline
                       withdraws remaining to buyer
                                    │
                                    ▼
                    ┌────────────────────────────────┐
                    │           CLOSED                │
                    │                                │
                    │  Remaining balance returned     │
                    │  to buyer                       │
                    └────────────────────────────────┘
```

---

## 3. Protocol Flow with Channels

### 3.1 Channel-Based x402 Payment

```
  CLIENT                     RESOURCE SERVER              FACILITATOR              HATHOR
    │                              │                           │                     │
    │  ┌─── SETUP (once) ──────────────────────────────────────────────────────┐    │
    │  │                           │                           │                │    │
    │  │ 1. Create X402Channel ──────────────────────────────────────────────▶  │    │
    │  │    (deposit 10 HTR)       │                           │         OPEN   │    │
    │  │◀── channelId ─────────────────────────────────────────────────────────  │    │
    │  └───────────────────────────────────────────────────────────────────────┘    │
    │                              │                           │                     │
    │  ┌─── REQUEST (instant, repeatable) ─────────────────────────────────────┐    │
    │  │                           │                           │                │    │
    │  │── GET /weather ─────────▶ │                           │                │    │
    │  │   + X-Payment {channelId} │                           │                │    │
    │  │                           │── verify channel ───────▶ │                │    │
    │  │                           │   {channelId, amount}     │── query state ▶│    │
    │  │                           │                           │◀─ {OPEN, bal} ─│    │
    │  │                           │◀── {valid: true} ──────── │                │    │
    │  │                           │                           │                │    │
    │  │◀── 200 + weather data ─── │                           │                │    │
    │  │                           │                           │                │    │
    │  │                           │── POST /x402/settle ────▶ │                │    │
    │  │                           │   {channelId, amount}     │── spend() ───▶ │    │
    │  │                           │                           │◀── txId ────── │    │
    │  │                           │◀── {success} ──────────── │                │    │
    │  └───────────────────────────────────────────────────────────────────────┘    │
    │                              │                           │                     │
    │  (repeat REQUEST as many times as channel has balance)   │                     │
```

### 3.2 What the Client Sends

The client includes a `X-Payment` header with `scheme: "hathor-channel"` (instead of `"hathor-escrow"`):

```json
{
  "scheme": "hathor-channel",
  "network": "hathor:privatenet",
  "payload": {
    "channelId": "000abc123...",
    "buyerAddress": "Wk1PQJJk..."
  }
}
```

Note: no `ncId` or `depositTxId` — the channel is already funded. The client just references it.

### 3.3 What the Server Advertises

The 402 response can advertise both escrow and channel support:

```json
{
  "accepts": [
    {
      "scheme": "hathor-escrow",
      "asset": "00",
      "amount": "100",
      "description": "Pay 1.00 HTR (single payment)",
      "extra": { "blueprintId": "...", "facilitatorAddress": "...", "facilitatorUrl": "..." }
    },
    {
      "scheme": "hathor-channel",
      "asset": "00",
      "amount": "100",
      "description": "Pay 1.00 HTR via channel (instant if pre-funded)",
      "extra": { "channelBlueprintId": "...", "facilitatorAddress": "...", "facilitatorUrl": "..." }
    }
  ]
}
```

### 3.4 Facilitator Changes

The facilitator needs two new behaviors for channels:

**Verify (channel mode):** Instead of checking that an escrow is LOCKED with the right amount, check that the channel is OPEN and has sufficient remaining balance (`total_deposited - total_spent >= amount`).

**Settle (channel mode):** Instead of calling `release()` (all-or-nothing), call `spend(amount, sellerAddress)` which deducts only the request amount from the channel.

---

## 4. Comparison: Escrow vs Channel

### 4.1 When to Use Which

| Use Case | Recommended | Why |
|----------|-------------|-----|
| One-off API call | Escrow | Simple, no setup overhead |
| Agent making many calls | Channel | Pay once, instant access |
| Unknown number of calls | Channel | Top up as needed |
| Different sellers per call | Channel | `spend()` takes seller address |
| Maximum trustlessness | Escrow | Funds locked per-request |
| Minimum latency | Channel | No per-request wait after setup |

### 4.2 Trust Model

Channels require slightly more trust in the facilitator than escrows:

- **Escrow:** The facilitator can only release to the seller specified at escrow creation. The buyer verified this before depositing.
- **Channel:** The facilitator specifies the seller at `spend()` time. The buyer trusts that the facilitator will spend to the correct seller.

This is acceptable because:
1. The facilitator is a known, trusted service (not anonymous)
2. Each `spend()` is on-chain and auditable
3. The buyer can close the channel at any time to recover remaining funds
4. The channel has a deadline — worst case, funds auto-refund

---

## 5. Implementation Plan

| # | Deliverable | Where |
|---|---|---|
| 1 | X402Channel blueprint | On-chain deploy |
| 2 | Facilitator: channel verify/settle | `facilitator.js` update |
| 3 | Resource server: advertise channel scheme | `resource-server.js` update |
| 4 | Client: channel creation + reuse | `client.js` update |
| 5 | dApp: channel management UI | `dapp/` update |
| 6 | Client SDK: channel support | `@hathor/x402-client` (future) |

---

## 6. Relationship to Other RFCs

- **0001-x402-support.md** — Base protocol. Channels extend it with a new scheme (`hathor-channel`).
- **0002-x402-client-sdk.md** — The client SDK should transparently manage channels: open one if the user makes repeated requests, reuse it, top up as needed.
- **0003-x402-server-middleware.md** — The server middleware should accept both schemes without the seller caring about the difference.
