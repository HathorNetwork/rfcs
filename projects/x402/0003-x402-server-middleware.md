- Feature Name: x402_server_middleware
- Start Date: 2026-04-02
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Andre Cardoso <andre@hathor.network>

# @hathor/x402-server — Server Middleware for x402 Payments on Hathor

---

## 1. Overview

This document describes the design for `@hathor/x402-server`, a TypeScript middleware package that enables any HTTP server to sell access to its resources via the x402 protocol on Hathor Network.

Adding x402 payment gating to an Express route should take **three lines of configuration**:

```typescript
import { hathorPaymentMiddleware } from '@hathor/x402-server';

app.use(hathorPaymentMiddleware({
  facilitatorUrl: 'https://facilitator.hathor.network',
  payTo: 'WXf4xPLBn7HUC7F1U2vY4J5zwpsDS12bT6',
  routes: {
    'GET /weather': { amount: '100', asset: '00' },
  },
}));
```

The middleware handles everything: returning 402 with the correct payment requirements, verifying payment via the facilitator, triggering settlement after serving the resource, and error handling.

### 1.1 Design Philosophy

**One middleware, any framework.** The core logic is framework-agnostic. We provide first-class adapters for Express, Fastify, and Hono, plus a generic adapter for any framework that supports request/response middleware.

**The seller never touches crypto.** The middleware abstracts away nano contracts, escrow verification, and settlement. The seller configures a price list and receives funds to their address. They don't need to understand how x402 works.

**Facilitator handles the hard parts.** Verification and settlement are delegated to the facilitator (a public service or the seller's own instance). The middleware is a thin HTTP layer that calls the facilitator and enforces the payment gate.

### 1.2 Goals

1. Provide middleware for Express, Fastify, and Hono
2. Declarative route-based pricing configuration
3. Multi-token support (accept HTR, stablecoins, or both)
4. Async settlement (don't block the response)
5. Framework-agnostic core with typed adapters
6. Zero crypto knowledge required from the seller

---

## 2. API Design

### 2.1 Express Middleware

```typescript
import express from 'express';
import { hathorPaymentMiddleware } from '@hathor/x402-server';

const app = express();

app.use(hathorPaymentMiddleware({
  // Required: facilitator service URL
  facilitatorUrl: 'https://facilitator.hathor.network',

  // Required: seller's receiving address
  payTo: 'WXf4xPLBn7HUC7F1U2vY4J5zwpsDS12bT6',

  // Required: route pricing
  routes: {
    'GET /weather': {
      amount: '100',         // 1.00 HTR
      asset: '00',
      description: 'Real-time weather data',
    },
    'GET /premium/*': {
      amount: '500',         // 5.00 HTR
      asset: '00',
      description: 'Premium content',
    },
    'POST /analyze': {
      // Accept multiple tokens
      pricing: [
        { amount: '100', asset: '00', description: 'Pay 1.00 HTR' },
        { amount: '1000', asset: '0000abc...', description: 'Or pay 10.00 hUSDC' },
      ],
    },
  },

  // Optional configuration (all have sensible defaults)
  network: 'hathor:mainnet',                    // default
  facilitatorAddress: 'WYyy...',                // auto-fetched from facilitator if omitted
  blueprintId: '0000abc...',                    // auto-fetched from facilitator if omitted
  deadlineSeconds: 300,                         // default: 5 minutes
  settlementMode: 'async',                      // 'async' (default) or 'sync'
  onPaymentVerified: (details) => {},           // callback after verification
  onSettlementComplete: (details) => {},        // callback after settlement
  onSettlementFailed: (error) => {},            // callback on settlement error
}));

// Routes are normal — middleware handles 402 gating
app.get('/weather', (req, res) => {
  res.json({ temperature: 28, city: 'Sao Paulo' });
});

app.get('/premium/:id', (req, res) => {
  res.json({ content: 'Premium article...' });
});
```

### 2.2 Fastify Plugin

```typescript
import Fastify from 'fastify';
import { hathorPaymentPlugin } from '@hathor/x402-server/fastify';

const app = Fastify();

app.register(hathorPaymentPlugin, {
  facilitatorUrl: 'https://facilitator.hathor.network',
  payTo: 'WXf4xPLBn7HUC7F1U2vY4J5zwpsDS12bT6',
  routes: {
    'GET /weather': { amount: '100', asset: '00' },
  },
});
```

### 2.3 Hono Middleware

```typescript
import { Hono } from 'hono';
import { hathorPayment } from '@hathor/x402-server/hono';

const app = new Hono();

app.use('/weather', hathorPayment({
  facilitatorUrl: 'https://facilitator.hathor.network',
  payTo: 'WXf4xPLBn7HUC7F1U2vY4J5zwpsDS12bT6',
  amount: '100',
  asset: '00',
}));
```

### 2.4 Configuration Types

```typescript
interface X402ServerConfig {
  /** URL of the facilitator service (public endpoint) */
  facilitatorUrl: string;

  /** Seller's Hathor address for receiving payments */
  payTo: string;

  /** Route pricing configuration */
  routes: Record<string, RoutePrice>;

  /** Network identifier. Default: 'hathor:mainnet' */
  network?: string;

  /**
   * Facilitator's Hathor address. If omitted, fetched from the
   * facilitator's discovery endpoint on startup.
   */
  facilitatorAddress?: string;

  /**
   * X402Escrow blueprint ID. If omitted, fetched from the
   * facilitator's discovery endpoint on startup.
   */
  blueprintId?: string;

  /** Escrow deadline in seconds. Default: 300 (5 minutes) */
  deadlineSeconds?: number;

  /**
   * Settlement mode:
   * - 'async' (default): settle after response, don't block
   * - 'sync': settle before response, block until confirmed
   */
  settlementMode?: 'async' | 'sync';

  /** Callback after payment verification succeeds */
  onPaymentVerified?: (details: VerifiedPayment) => void;

  /** Callback after settlement completes */
  onSettlementComplete?: (details: SettlementResult) => void;

  /** Callback when settlement fails */
  onSettlementFailed?: (error: SettlementError) => void;
}

/** Simple single-token pricing */
interface SinglePrice {
  amount: string;
  asset: string;
  description?: string;
}

/** Multi-token pricing */
interface MultiPrice {
  pricing: SinglePrice[];
}

type RoutePrice = SinglePrice | MultiPrice;

interface VerifiedPayment {
  ncId: string;
  asset: string;
  amount: number;
  buyerAddress: string;
  route: string;
  method: string;
}

interface SettlementResult {
  ncId: string;
  txId: string;
  asset: string;
  amount: number;
}

interface SettlementError {
  ncId: string;
  error: string;
  /** Whether the settlement should be retried */
  retryable: boolean;
}
```

---

## 3. Route Matching

### 3.1 Pattern Syntax

Routes are specified as `METHOD /path` strings. The path supports:

| Pattern | Example | Matches |
|---------|---------|---------|
| Exact | `GET /weather` | Only `GET /weather` |
| Wildcard suffix | `GET /premium/*` | `GET /premium/123`, `GET /premium/a/b/c` |
| Named param | `GET /users/:id` | `GET /users/123`, `GET /users/abc` |
| All methods | `* /api/data` | Any method to `/api/data` |

### 3.2 Matching Priority

1. Exact match (method + path)
2. Named param match
3. Wildcard match
4. No match → `next()` (pass through, no 402)

### 3.3 Dynamic Pricing

For routes where the price depends on the request (e.g., token amount varies by resource size), use a function:

```typescript
routes: {
  'GET /data/:id': (req) => {
    const size = lookupDataSize(req.params.id);
    return { amount: String(size * 10), asset: '00' };
  },
},
```

```typescript
type RoutePrice = SinglePrice | MultiPrice | ((req: Request) => SinglePrice | MultiPrice);
```

---

## 4. Internal Flow

### 4.1 Middleware Request Lifecycle

```
Incoming Request
    │
    ├── 1. Match route against config.routes
    │       └── No match? → next() (pass through)
    │
    ├── 2. Check for X-Payment header
    │       └── No header? → Return 402 + payment requirements
    │
    ├── 3. Parse X-Payment header
    │       └── Invalid JSON? → Return 400
    │
    ├── 4. Call facilitator: POST /x402/verify
    │       ├── Build paymentRequirements from route config
    │       ├── Send paymentPayload + paymentRequirements
    │       └── Not valid? → Return 402 + error details
    │
    ├── 5. Attach payment info to request (req.x402)
    │
    ├── 6. Call next() → route handler serves the resource
    │
    ├── 7. After response: trigger settlement
    │       ├── async mode: fire-and-forget POST /x402/settle
    │       └── sync mode: await settlement, then respond
    │
    └── 8. Call onSettlementComplete or onSettlementFailed
```

### 4.2 402 Response Format

When no payment header is present, the middleware returns:

```json
{
  "accepts": [
    {
      "scheme": "hathor-escrow",
      "network": "hathor:mainnet",
      "asset": "00",
      "amount": "100",
      "resource": "https://api.example.com/weather",
      "description": "Pay 1.00 HTR",
      "mimeType": "application/json",
      "payTo": "WXf4xPLBn7HUC7F1U2vY4J5zwpsDS12bT6",
      "maxTimeoutSeconds": 300,
      "extra": {
        "facilitatorUrl": "https://facilitator.hathor.network",
        "facilitatorAddress": "WYyy...",
        "blueprintId": "0000abc...",
        "deadlineSeconds": 300
      }
    }
  ],
  "version": "1"
}
```

For multi-token routes, the `accepts` array contains one entry per token option.

### 4.3 Settlement

After the route handler sends its response, the middleware triggers settlement:

```typescript
async function settle(facilitatorUrl: string, paymentPayload: PaymentPayload): Promise<SettlementResult> {
  const resp = await fetch(`${facilitatorUrl}/x402/settle`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ paymentPayload }),
  });
  return resp.json();
}
```

**Async mode (default):** Settlement runs after the response is sent. If it fails, the `onSettlementFailed` callback is called. The seller can implement retry logic there.

**Sync mode:** Settlement runs before the response. The response includes the settlement transaction ID. Use this when the client needs proof of settlement.

### 4.4 Request Decoration

After successful verification, the middleware attaches payment info to the request:

```typescript
// Express
app.get('/weather', (req, res) => {
  console.log(req.x402.ncId);         // Escrow contract ID
  console.log(req.x402.buyerAddress); // Who paid
  console.log(req.x402.amount);       // How much
  console.log(req.x402.asset);        // Which token
  res.json({ ... });
});
```

This allows route handlers to customize responses based on payment details (e.g., return more data for higher payments).

---

## 5. Facilitator Discovery

### 5.1 Auto-Configuration

If `facilitatorAddress` or `blueprintId` are omitted from the config, the middleware fetches them from the facilitator on startup:

```
GET {facilitatorUrl}/x402/info
```

```json
{
  "facilitatorAddress": "WYyy...",
  "blueprintId": "0000abc...",
  "network": "hathor:mainnet",
  "supportedTokens": ["00", "0000abc..."],
  "version": "1.0.0"
}
```

This simplifies configuration — the seller only needs the `facilitatorUrl` and `payTo` address. Everything else is discovered automatically.

### 5.2 Health Checks

The middleware periodically pings the facilitator to ensure it's available:

```
GET {facilitatorUrl}/x402/health
```

If the facilitator is unreachable, the middleware can:
- Continue returning 402s (clients can't verify) — **default**
- Pass requests through unpaid (graceful degradation) — opt-in via `fallbackMode: 'passthrough'`
- Return 503 (service unavailable) — opt-in via `fallbackMode: 'reject'`

---

## 6. Advanced Features

### 6.1 Per-Route Facilitator Override

Different routes can use different facilitators:

```typescript
routes: {
  'GET /weather': {
    amount: '100',
    asset: '00',
    facilitatorUrl: 'https://my-own-facilitator.com',
  },
  'GET /premium/*': {
    amount: '500',
    asset: '00',
    // Uses the global facilitatorUrl
  },
}
```

### 6.2 Free Tier / Rate Limiting

The middleware can integrate with rate limiting to offer free requests before requiring payment:

```typescript
routes: {
  'GET /weather': {
    amount: '100',
    asset: '00',
    freeRequests: {
      count: 10,        // 10 free requests
      window: '1h',      // per hour
      keyFn: (req) => req.ip,  // per IP
    },
  },
}
```

When free requests are exhausted, the middleware returns 402.

### 6.3 Dynamic Address Generation

For sellers who want a unique receiving address per transaction (for easier accounting), the middleware can call the seller's private headless wallet:

```typescript
const middleware = hathorPaymentMiddleware({
  // Instead of a static payTo address, use a headless wallet
  addressSource: {
    type: 'headless',
    url: 'http://localhost:8000',  // seller's private headless
    walletId: 'seller-wallet',
  },
  // ...
});
```

Each 402 response gets a fresh address from `GET /wallet/address`.

### 6.4 Webhook Notifications

For sellers who need to track payments externally (e.g., update a database, send a notification):

```typescript
onPaymentVerified: async (details) => {
  await db.payments.insert({
    ncId: details.ncId,
    buyer: details.buyerAddress,
    amount: details.amount,
    asset: details.asset,
    route: details.route,
    timestamp: new Date(),
  });
},

onSettlementComplete: async (details) => {
  await slack.notify(`Payment settled: ${details.txId}`);
},
```

---

## 7. Package Structure

```
@hathor/x402-server/
├── src/
│   ├── index.ts              # Express adapter (default export)
│   ├── core.ts               # Framework-agnostic core logic
│   ├── adapters/
│   │   ├── express.ts        # Express middleware adapter
│   │   ├── fastify.ts        # Fastify plugin adapter
│   │   └── hono.ts           # Hono middleware adapter
│   ├── route-matcher.ts      # Route pattern matching
│   ├── facilitator-client.ts # Facilitator HTTP client (verify, settle, info)
│   ├── payment-builder.ts    # 402 response builder
│   └── types.ts              # All TypeScript interfaces
├── package.json
├── tsconfig.json
└── README.md
```

**Dependencies:**
- None for core
- Framework types as optional peer dependencies

**Exports:**

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./fastify": "./dist/adapters/fastify.js",
    "./hono": "./dist/adapters/hono.js",
    "./core": "./dist/core.js"
  }
}
```

---

## 8. Security Considerations

### 8.1 Facilitator Trust

The middleware trusts the facilitator's verification response. If the facilitator is compromised, it could falsely verify payments. Mitigation:
- Use Hathor's public facilitator or run your own
- The facilitator only verifies on-chain state — a compromised facilitator can lie about state but cannot steal funds
- Settlement still requires a valid on-chain transaction

### 8.2 Replay Protection

Each escrow contract has a unique `ncId`. The middleware does not cache or track `ncId`s — it delegates replay protection to the facilitator. The facilitator ensures:
- The escrow is in `LOCKED` state (a released/refunded escrow can't be reused)
- The escrow's `resource_url` matches (optional additional check)

### 8.3 Header Tampering

The `X-Payment` header is not signed — it's just a pointer to the on-chain escrow. Tampering with it (changing the `ncId`) would point to a different escrow, which the facilitator would reject if it doesn't match the payment requirements. The on-chain state is the source of truth, not the header.

### 8.4 Settlement Failures

If settlement fails after the resource is served, the funds remain locked in the escrow. The middleware reports this via `onSettlementFailed`. The seller should implement retry logic:

```typescript
onSettlementFailed: async (error) => {
  if (error.retryable) {
    await settlementRetryQueue.add(error.ncId);
  } else {
    await alerts.critical(`Non-retryable settlement failure: ${error.ncId}`);
  }
},
```

Worst case: the escrow deadline expires and the buyer gets refunded. The seller loses the resource cost but never loses funds they already had.

---

## 9. Facilitator API Extensions

This middleware design assumes the facilitator exposes an additional endpoint beyond what the base x402 RFC specifies:

### 9.1 `GET /x402/info` — Discovery Endpoint

```json
{
  "facilitatorAddress": "WYyy...",
  "blueprintId": "0000abc...",
  "network": "hathor:mainnet",
  "supportedTokens": ["00", "0000abc..."],
  "maxEscrowAmount": 100000,
  "version": "1.0.0"
}
```

This allows the middleware to auto-configure itself by querying the facilitator. The facilitator RFC (0001) should be updated to include this endpoint.

### 9.2 `GET /x402/health` — Health Check

```json
{
  "status": "ok",
  "walletConnected": true,
  "fullnodeConnected": true,
  "uptime": 86400
}
```

---

## 10. Comparison with Existing x402 Server Libraries

| Feature | coinbase/x402 (EVM) | @hathor/x402-server (this design) |
|---------|---------------------|-----------------------------------|
| Framework support | Express only | Express, Fastify, Hono, generic |
| Payment mechanism | EIP-3009 signed message | Nano contract escrow |
| Verification | On-chain allowance check | Facilitator state query |
| Settlement | Facilitator submits tx | Facilitator calls release() |
| Multi-token | Yes (ERC-20s) | Yes (any Hathor token) |
| Route matching | Path prefix | Glob, named params, wildcard |
| Dynamic pricing | No | Yes (function-based) |
| Free tier | No | Built-in rate limiting |
| Address generation | Static | Static or dynamic via headless |

---

## 11. Implementation Plan

| Phase | Deliverable | Timeline |
|-------|------------|----------|
| 1 | Core logic + Express adapter | Week 1-2 |
| 2 | Route matching, 402 builder, facilitator client | Week 2-3 |
| 3 | Fastify and Hono adapters | Week 3-4 |
| 4 | Advanced features (dynamic pricing, free tier, address gen) | Week 4-5 |
| 5 | Tests (unit + integration with hathor-forge + x402-poc) | Week 5-6 |
| 6 | npm publish `@hathor/x402-server` | Week 6 |

---

## 12. Relationship to Other RFCs

- **0001-x402-support.md** — The base protocol. This middleware implements the resource server role.
- **0002-x402-client-sdk.md** — The client counterpart. Client SDK wraps `fetch` for payers; this middleware wraps Express/Fastify for sellers.
- **Facilitator plugin** — Described in 0001. The middleware delegates to the facilitator; it does not implement facilitator logic.
