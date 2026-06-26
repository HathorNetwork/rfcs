- Feature Name: x402_client_sdk
- Start Date: 2026-04-02
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Andre Cardoso <andre@hathor.network>

# @hathor/x402-client — Client SDK for x402 Payments on Hathor

---

## 1. Overview

This document describes the design for `@hathor/x402-client`, a TypeScript SDK that enables any programmatic client (AI agent, script, backend service) to automatically pay for x402-gated HTTP resources on Hathor Network.

The SDK wraps the standard `fetch` API. When a server returns HTTP 402, the wrapper transparently:

1. Parses the payment requirements
2. Selects a payment option (token, amount)
3. Creates an X402Escrow nano contract via the client's wallet backend
4. Waits for on-chain confirmation
5. Retries the request with the payment proof

The developer sees a single `fetch` call. The x402 protocol is invisible.

### 1.1 Design Philosophy

**Zero protocol knowledge required.** A developer using this SDK should never need to understand escrow contracts, nano contract creation, token UIDs, or facilitator APIs. They call `fetch`, they get data, their wallet pays.

**Two wallet backends, one API.** The SDK supports two wallet backends:
- **hathor-wallet-headless** — for AI agents and servers that run their own headless instance
- **@hathor/wallet-lib** — for lightweight Node.js apps that embed the wallet directly

The developer picks a backend at configuration time. The `fetch` wrapper API is identical.

**Fail-safe defaults.** If the SDK cannot pay (insufficient balance, no matching token, wallet offline), it returns the original 402 response — never swallows errors silently.

### 1.2 Goals

1. Provide a `fetch` wrapper that handles 402 → pay → retry automatically
2. Support both headless wallet and wallet-lib backends
3. Support multi-token payment selection
4. TypeScript-first with full type safety
5. Zero runtime dependencies beyond the wallet backend

---

## 2. API Design

### 2.1 Primary API — `wrapFetchWithHathorPayment`

```typescript
import { wrapFetchWithHathorPayment } from '@hathor/x402-client';

const fetch402 = wrapFetchWithHathorPayment(fetch, {
  walletBackend: {
    type: 'headless',
    url: 'http://localhost:8000',
    walletId: 'my-agent-wallet',
  },
  // Optional configuration
  tokenPreference: ['00'],                // prefer HTR, then any
  maxAmountPerRequest: 1000,              // safety limit: 10.00 HTR max
  onPayment: (details) => console.log(details), // payment callback
});

// Usage — identical to fetch()
const response = await fetch402('https://api.example.com/weather');
const data = await response.json();
```

**Function signature:**

```typescript
function wrapFetchWithHathorPayment(
  fetchFn: typeof fetch,
  config: X402ClientConfig,
): typeof fetch;
```

The returned function has the **exact same signature as `fetch`**. It can be used as a drop-in replacement anywhere `fetch` is used — including in HTTP client libraries, MCP tool implementations, and AI agent frameworks.

### 2.2 Configuration

```typescript
interface X402ClientConfig {
  /** Wallet backend configuration */
  walletBackend: HeadlessBackend | WalletLibBackend;

  /**
   * Token preference order. The SDK picks the first token in this list
   * that appears in the server's `accepts` array.
   * Default: ['00'] (prefer HTR)
   */
  tokenPreference?: string[];

  /**
   * Maximum amount (in smallest unit) the SDK will pay per request.
   * If the server asks for more, the 402 is returned as-is.
   * Default: Infinity (no limit)
   */
  maxAmountPerRequest?: number;

  /**
   * Callback fired after each successful payment.
   * Useful for logging, analytics, budget tracking.
   */
  onPayment?: (details: PaymentDetails) => void;

  /**
   * Callback fired when a payment fails or is skipped.
   */
  onPaymentError?: (error: PaymentError) => void;

  /**
   * Maximum time (ms) to wait for escrow tx confirmation.
   * Default: 60000 (60 seconds)
   */
  confirmationTimeoutMs?: number;

  /**
   * Custom request hash function. By default, hashes method + URL.
   * Override to include request body, headers, etc.
   */
  requestHashFn?: (url: string, init?: RequestInit) => string;
}

interface HeadlessBackend {
  type: 'headless';
  /** URL of the private headless wallet instance */
  url: string;
  /** Wallet ID within the headless instance */
  walletId: string;
  /** Optional: address to use for escrow deposits. If omitted, uses the wallet's current address. */
  address?: string;
}

interface WalletLibBackend {
  type: 'wallet-lib';
  /** A started HathorWallet instance from @hathor/wallet-lib */
  wallet: HathorWallet;
  /** Optional: address index to use */
  addressIndex?: number;
}
```

### 2.3 Payment Details Callback

```typescript
interface PaymentDetails {
  /** The URL that was paid for */
  url: string;
  /** The nano contract ID (= escrow deposit tx hash) */
  ncId: string;
  /** Token UID used for payment */
  asset: string;
  /** Amount paid (smallest unit) */
  amount: number;
  /** Seller address */
  payTo: string;
  /** Time from 402 received to resource received (ms) */
  totalLatencyMs: number;
  /** Time waiting for tx confirmation (ms) */
  confirmationLatencyMs: number;
}

interface PaymentError {
  url: string;
  reason: 'insufficient_balance' | 'no_matching_token' | 'wallet_error'
        | 'confirmation_timeout' | 'payment_rejected' | 'amount_exceeds_limit';
  details: string;
}
```

### 2.4 Alternative API — `payForResource` (Low-Level)

For developers who need more control:

```typescript
import { payForResource } from '@hathor/x402-client';

// Step 1: You already have the 402 response
const paymentRequired = await response.json();

// Step 2: Pay
const paymentProof = await payForResource(walletBackend, paymentRequired, {
  tokenPreference: ['00'],
});

// Step 3: Retry yourself
const paidResponse = await fetch(url, {
  headers: { 'X-Payment': JSON.stringify(paymentProof) },
});
```

```typescript
async function payForResource(
  backend: HeadlessBackend | WalletLibBackend,
  paymentRequired: PaymentRequired402,
  options?: { tokenPreference?: string[] },
): Promise<PaymentPayload>;
```

---

## 3. Internal Flow

### 3.1 Fetch Wrapper Logic

```
fetch402(url, init)
    │
    ├── 1. Call original fetch(url, init)
    │
    ├── 2. Response status !== 402?
    │       └── YES → return response as-is
    │
    ├── 3. Parse 402 body → PaymentRequired402
    │
    ├── 4. Select payment option from accepts[]
    │       ├── Filter by tokenPreference order
    │       ├── Check amount <= maxAmountPerRequest
    │       └── No match? → return original 402 response
    │
    ├── 5. Create escrow nano contract
    │       ├── Headless: POST /wallet/nano-contracts/create
    │       └── WalletLib: NanoContractTransactionBuilder
    │
    ├── 6. Wait for tx confirmation
    │       ├── Poll fullnode /transaction?id={ncId}
    │       └── Timeout? → call onPaymentError, return 402
    │
    ├── 7. Build X-Payment header
    │       └── { scheme, network, payload: { ncId, depositTxId, buyerAddress } }
    │
    ├── 8. Retry original fetch(url, init + X-Payment header)
    │
    ├── 9. Response status === 200?
    │       ├── YES → call onPayment callback, return response
    │       └── NO  → call onPaymentError, return response
    │
    └── 10. Return response
```

### 3.2 Token Selection Algorithm

```
Given: accepts[] from 402, tokenPreference[] from config

1. For each token in tokenPreference:
     Find first entry in accepts[] where entry.asset === token
     If found AND entry.amount <= maxAmountPerRequest:
       → select this entry

2. If no preference match:
     Find first entry in accepts[] where entry.amount <= maxAmountPerRequest
     → select this entry (any token the wallet can pay)

3. If no entry within budget:
     → return null (skip payment, return original 402)
```

### 3.3 Headless Backend Implementation

```typescript
async function createEscrowViaHeadless(
  config: HeadlessBackend,
  option: PaymentOption,
): Promise<{ ncId: string }> {
  // Get buyer address if not configured
  const address = config.address
    ?? await getAddress(config.url, config.walletId);

  const deadline = Math.floor(Date.now() / 1000) + option.extra.deadlineSeconds;
  const requestHash = hashRequest(option.resource);

  const result = await post(`${config.url}/wallet/nano-contracts/create`, {
    blueprint_id: option.extra.blueprintId,
    address,
    data: {
      args: [
        option.payTo,                      // seller
        option.extra.facilitatorAddress,    // facilitator
        option.asset,                      // token_uid
        deadline,                          // deadline
        option.resource,                   // resource_url
        requestHash,                       // request_hash
      ],
      actions: [{
        type: 'deposit',
        token: option.asset,
        amount: parseInt(option.amount),
      }],
    },
  }, { 'X-Wallet-Id': config.walletId });

  if (!result.success) {
    throw new PaymentError('wallet_error', result.message);
  }

  return { ncId: result.hash };
}
```

### 3.4 Wallet-Lib Backend Implementation

```typescript
async function createEscrowViaWalletLib(
  config: WalletLibBackend,
  option: PaymentOption,
): Promise<{ ncId: string }> {
  const wallet = config.wallet;
  const address = await wallet.getAddressAtIndex(config.addressIndex ?? 0);
  const deadline = Math.floor(Date.now() / 1000) + option.extra.deadlineSeconds;

  // Build nano contract transaction using wallet-lib
  const tx = new NanoContractTransactionBuilder()
    .setBlueprintId(option.extra.blueprintId)
    .setMethod('initialize')
    .setCallerAddress(address)
    .addArgument('seller', 'Address', option.payTo)
    .addArgument('facilitator', 'Address', option.extra.facilitatorAddress)
    .addArgument('token_uid', 'TokenUid', option.asset)
    .addArgument('deadline', 'Timestamp', deadline)
    .addArgument('resource_url', 'str', option.resource)
    .addArgument('request_hash', 'str', hashRequest(option.resource))
    .addAction({ type: 'deposit', token: option.asset, amount: parseInt(option.amount) })
    .build();

  const result = await wallet.sendTransaction(tx);
  return { ncId: result.hash };
}
```

### 3.5 Confirmation Polling

```typescript
async function waitForConfirmation(
  fullnodeUrl: string,
  txId: string,
  timeoutMs: number,
): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    const resp = await fetch(`${fullnodeUrl}/v1a/transaction?id=${txId}`);
    const data = await resp.json();
    if (data.success && data.meta?.first_block) {
      return; // confirmed
    }
    await sleep(1000);
  }
  throw new PaymentError('confirmation_timeout', `TX ${txId} not confirmed within ${timeoutMs}ms`);
}
```

---

## 4. Integration Examples

### 4.1 AI Agent (MCP Tool)

```typescript
import { wrapFetchWithHathorPayment } from '@hathor/x402-client';

// Agent's wallet is running on localhost
const fetch402 = wrapFetchWithHathorPayment(fetch, {
  walletBackend: {
    type: 'headless',
    url: 'http://localhost:8000',
    walletId: 'agent-wallet',
  },
  maxAmountPerRequest: 500, // max 5 HTR per call
  onPayment: (d) => console.log(`Paid ${d.amount} for ${d.url}`),
});

// MCP tool implementation
async function getWeather(city: string) {
  const resp = await fetch402(`https://weather-api.example.com/v1/${city}`);
  return resp.json();
}
```

### 4.2 Backend Service with Budget Tracking

```typescript
import { wrapFetchWithHathorPayment } from '@hathor/x402-client';

let totalSpent = 0;
const BUDGET_LIMIT = 100000; // 1000 HTR

const fetch402 = wrapFetchWithHathorPayment(fetch, {
  walletBackend: {
    type: 'headless',
    url: process.env.HEADLESS_URL,
    walletId: 'service-wallet',
  },
  tokenPreference: ['00', process.env.HUSDC_TOKEN_UID],
  onPayment: (d) => {
    totalSpent += d.amount;
    if (totalSpent > BUDGET_LIMIT) {
      console.warn('Budget limit approaching!');
    }
  },
});
```

### 4.3 Lightweight Client (wallet-lib, no headless)

```typescript
import { HathorWallet } from '@hathor/wallet-lib';
import { wrapFetchWithHathorPayment } from '@hathor/x402-client';

const wallet = new HathorWallet({
  seed: process.env.WALLET_SEED,
  server: 'https://node.hathor.network/v1a/',
  network: 'mainnet',
});
await wallet.start();

const fetch402 = wrapFetchWithHathorPayment(fetch, {
  walletBackend: { type: 'wallet-lib', wallet },
});

const data = await fetch402('https://api.example.com/data').then(r => r.json());
```

---

## 5. Error Handling

### 5.1 Principle: Never Swallow Errors

The SDK never hides failures. If it can't pay, the developer gets the original 402 response (or the error response from the retry). The `onPaymentError` callback provides structured diagnostics.

### 5.2 Error Scenarios

| Scenario | SDK Behavior |
|----------|-------------|
| Server returns 402, no matching token | Return original 402 response, call `onPaymentError('no_matching_token')` |
| Server returns 402, amount > `maxAmountPerRequest` | Return original 402 response, call `onPaymentError('amount_exceeds_limit')` |
| Wallet has insufficient balance | Return original 402 response, call `onPaymentError('insufficient_balance')` |
| Wallet-headless is unreachable | Throw (network error — same as `fetch` throwing) |
| Escrow tx not confirmed within timeout | Return original 402 response, call `onPaymentError('confirmation_timeout')` |
| Server rejects payment on retry (402 again) | Return the retry's 402 response, call `onPaymentError('payment_rejected')` |
| Server returns 200 on retry | Return 200 response, call `onPayment(details)` |
| Server returns non-402 on initial request | Return response as-is (no payment logic triggered) |

### 5.3 Idempotency Concern

If the escrow is created but the retry fails (network error), the funds are locked in the contract. The SDK does **not** automatically refund — the escrow's deadline handles this. The `onPaymentError` callback reports the `ncId` so the developer can track or manually refund.

For production use, developers should implement retry logic around the paid fetch, reusing the same `ncId` for retries (the SDK could support this with a `reusePayment` option in a future version).

---

## 6. Package Structure

```
@hathor/x402-client/
├── src/
│   ├── index.ts              # Public API exports
│   ├── wrap-fetch.ts         # wrapFetchWithHathorPayment implementation
│   ├── pay.ts                # payForResource low-level API
│   ├── token-selection.ts    # Token preference matching
│   ├── backends/
│   │   ├── headless.ts       # hathor-wallet-headless backend
│   │   └── wallet-lib.ts     # @hathor/wallet-lib backend
│   ├── confirmation.ts       # TX confirmation polling
│   └── types.ts              # All TypeScript interfaces
├── package.json
├── tsconfig.json
└── README.md
```

**Dependencies:**
- None for core (uses native `fetch`)
- `@hathor/wallet-lib` as optional peer dependency (only for wallet-lib backend)

---

## 7. Security Considerations

### 7.1 Private Wallet Access

The SDK requires access to a wallet backend (headless URL or wallet-lib instance). This is the agent's **private** wallet — it must never be exposed to the internet. The SDK only makes outbound calls to:
- The wallet backend (localhost)
- The public resource server (internet)
- The public full node (for confirmation polling, only if wallet-lib backend)

### 7.2 Amount Limits

The `maxAmountPerRequest` config is a critical safety feature. Without it, a malicious server could return a 402 asking for the wallet's entire balance. The SDK enforces this limit before creating the escrow.

### 7.3 Request Hash

The `request_hash` field in the escrow ties the payment to a specific request. This prevents a server from accepting payment for one resource and serving a different one. The default hash function uses the HTTP method + URL. Developers can override this to include the request body, headers, or any other context.

### 7.4 Escrow Deadline

The SDK sets the escrow deadline based on `deadlineSeconds` from the server's 402 response. This is the maximum time the funds are locked. If the server disappears after collecting payment, the buyer gets an automatic refund after the deadline.

---

## 8. Future Extensions

### 8.1 Payment Caching / Reuse

For APIs where the same resource is requested multiple times, the SDK could cache the `ncId` and reuse it on subsequent requests (if the escrow is still LOCKED and the server accepts cached payments). This eliminates per-request escrow creation.

### 8.2 Pre-funded Escrow Channels

A higher-level API that creates a single large escrow and draws against it for multiple requests. This is a protocol-level extension (requires a new blueprint) but the SDK should be designed to support it transparently.

### 8.3 Balance Checking

Before creating an escrow, the SDK could check the wallet's balance and fail fast if insufficient. This avoids waiting for a failed transaction. The headless backend supports `GET /wallet/balance` for this.

### 8.4 Multi-Request Batching

For agents that make multiple paid requests concurrently, the SDK could batch escrow creation to reduce transaction overhead.

---

## 9. Implementation Plan

| Phase | Deliverable | Timeline |
|-------|------------|----------|
| 1 | Core `wrapFetchWithHathorPayment` with headless backend | Week 1-2 |
| 2 | Token selection, amount limits, callbacks | Week 2-3 |
| 3 | wallet-lib backend | Week 3-4 |
| 4 | Tests (unit + integration with hathor-forge) | Week 4-5 |
| 5 | npm publish `@hathor/x402-client` | Week 5 |
