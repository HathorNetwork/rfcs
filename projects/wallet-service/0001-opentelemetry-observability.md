- Feature Name: opentelemetry_observability
- Start Date: 2026-03-05
- RFC PR:
- Hathor Issue:
- Author: Andre Cardoso <andre@hathor.network>

# Summary
[summary]: #summary

Add distributed tracing, metrics, and structured logging to the wallet-service monorepo using OpenTelemetry (OTel), the CNCF-graduated vendor-neutral observability standard. This gives us end-to-end visibility into request latencies, database query durations, and cross-component call chains across the daemon (K8s) and wallet-service (Lambda) packages — without locking us into any specific backend.

# Motivation
[motivation]: #motivation

We are currently blind to performance characteristics of the wallet-service. When something is slow, we have no way to know *which* function or query is the bottleneck. Our observability today is limited to:

- **Winston text logs** — no structured latency data, no correlation across components.
- **One Prometheus metric** (`best_block_height`) — tells us nothing about request performance.
- **CloudWatch alarms** — coarse Lambda-level metrics (duration, errors), no breakdown by handler logic or DB query.
- **No distributed tracing** — a request that flows from API Gateway → Lambda → MySQL → Redis → SQS is invisible as a whole.

This means:

1. **Incident diagnosis is slow.** When users report "the wallet is slow," we grep logs and guess. We cannot answer "which DB query took 3 seconds?" or "is Redis the bottleneck?"
2. **Performance regressions go unnoticed.** Without latency histograms, we cannot track p50/p95/p99 over time.
3. **Capacity planning is guesswork.** We do not know the actual load profile of our DB pool or Lambda concurrency.

The expected outcome is a system where any developer can pull up a trace for a specific API call and see the full breakdown: middleware time, handler logic, each DB query, Redis calls, SQS publishes — with durations attached to each span.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## What changes for developers

### Automatic instrumentation (zero code changes)

Most observability comes for free. By loading OTel auto-instrumentation at startup, every outgoing MySQL query, HTTP request, Redis call, and AWS SDK call is automatically traced. Developers do not need to modify existing handler code.

**Daemon** — add a `tracing.ts` file that initializes the OTel SDK before any other imports:

```typescript
// packages/daemon/src/tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  serviceName: 'wallet-service-daemon',
  traceExporter: new OTLPTraceExporter(),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

Then in the entrypoint: `import './tracing';` before all other imports.

**Wallet-service Lambdas** — use the ADOT (AWS Distro for OpenTelemetry) Lambda layer. This wraps the Lambda handler automatically and exports traces to X-Ray or any OTLP endpoint. Configuration is via environment variables — no code changes to individual handlers.

### Adding custom spans (when needed)

For business-critical paths, developers can add custom spans to get more granular visibility:

```typescript
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('wallet-service');

async function handleVoidedTx(tx: Transaction) {
  return tracer.startActiveSpan('handleVoidedTx', async (span) => {
    span.setAttribute('tx.id', tx.tx_id);
    span.setAttribute('tx.inputs_count', tx.inputs.length);
    try {
      // ... existing logic
    } finally {
      span.end();
    }
  });
}
```

### Viewing traces

Developers query traces in the chosen backend (see Reference-level explanation). A typical workflow:

1. User reports a slow API call.
2. Developer searches traces by `http.route = /wallet/addresses` and sorts by duration.
3. The trace shows: Lambda cold start (400ms) → middleware (5ms) → MySQL query (2.3s) → response (1ms).
4. Root cause: a missing index on the query that took 2.3s.

## What changes operationally

- A collector sidecar (or ADOT Lambda layer) runs alongside each component.
- Traces are exported to a backend (X-Ray, Grafana Tempo, or Jaeger — decision in Unresolved Questions).
- Dashboards show p50/p95/p99 latencies per endpoint, per DB query, per external call.
- Alerts fire on latency SLO breaches (e.g., p99 > 5s for any API endpoint).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Architecture overview

```
┌─────────────────────────┐     ┌──────────────────────────────┐
│  Daemon (K8s Pod)       │     │  Wallet-Service (Lambda)     │
│                         │     │                              │
│  tracing.ts (SDK init)  │     │  ADOT Lambda Layer           │
│       │                 │     │       │                      │
│  auto-instrumentation:  │     │  auto-instrumentation:       │
│  - mysql2               │     │  - mysql2                    │
│  - ws (WebSocket)       │     │  - aws-sdk (SQS, Lambda)     │
│  - http                 │     │  - http                      │
│       │                 │     │  - ioredis                   │
│  OTLPExporter ──────────┼──┐  │       │                      │
│                         │  │  │  OTLPExporter ───────────────┼──┐
└─────────────────────────┘  │  └──────────────────────────────┘  │
                             │                                     │
                             ▼                                     ▼
                    ┌──────────────────────┐
                    │  OTel Collector /    │
                    │  ADOT Collector      │
                    │                      │
                    │  → Backend (X-Ray /  │
                    │    Tempo / Jaeger)   │
                    └──────────────────────┘
```

## Package dependencies

All packages are from the `@opentelemetry` namespace (JS SDK 2.0, stable tracing API):

**Core (both packages):**
- `@opentelemetry/api` — vendor-neutral tracing API (singleton, one version per process)
- `@opentelemetry/sdk-node` — SDK initialization, span processors, resource detection

**Auto-instrumentation:**
- `@opentelemetry/auto-instrumentations-node` — meta-package that includes instrumentations for mysql2, http, ioredis, aws-sdk, and more

**Exporters (choose one per deployment):**
- `@opentelemetry/exporter-trace-otlp-http` — sends spans to any OTLP-compatible collector
- `@opentelemetry/exporter-trace-otlp-grpc` — gRPC alternative (lower overhead, higher throughput)

**Lambda-specific:**
- AWS ADOT Lambda layer (managed by AWS, no npm dependency needed)
- `disableAwsContextPropagation: true` must be set unless using X-Ray as the backend

## Daemon instrumentation

The daemon is a long-running K8s pod. OTel SDK initializes once at startup.

**Initialization (`packages/daemon/src/tracing.ts`):**

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-node';
import { Resource } from '@opentelemetry/resources';

const sdk = new NodeSDK({
  resource: new Resource({
    'service.name': process.env.OTEL_SERVICE_NAME || 'wallet-service-daemon',
    'service.version': process.env.SERVICE_VERSION || 'unknown',
    'deployment.environment': process.env.STAGE || 'local',
  }),
  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
    }),
    {
      maxQueueSize: 2048,
      maxExportBatchSize: 512,
      scheduledDelayMillis: 5000,
    },
  ),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false },
      '@opentelemetry/instrumentation-dns': { enabled: false },
    }),
  ],
});

sdk.start();

process.on('SIGTERM', () => sdk.shutdown());
```

Key decisions:
- `BatchSpanProcessor` (not `SimpleSpanProcessor`) for production — buffers spans and exports in batches to minimize overhead.
- Disable `fs` and `dns` instrumentations — they generate noise without value for our use case.
- Graceful shutdown on SIGTERM to flush pending spans.

**Custom spans for critical paths:**

Add manual spans to the event handlers in `services/` (e.g., `handleVoidedTx`, `handleVertexAccepted`) and to the SyncMachine transitions. These are the paths where we most need latency visibility.

## Wallet-service Lambda instrumentation

**Option A — ADOT Lambda Layer (recommended):**

Add the ADOT layer ARN to the Serverless Framework configuration:

```yaml
# serverless.yml
provider:
  layers:
    - arn:aws:lambda:${region}:901920570463:layer:aws-otel-nodejs-amd64-ver-1-18-1:1
  environment:
    AWS_LAMBDA_EXEC_WRAPPER: /opt/otel-handler
    OPENTELEMETRY_COLLECTOR_CONFIG_FILE: /var/task/collector.yaml
```

This wraps every Lambda handler automatically. No code changes to individual functions.

**Option B — Manual SDK init (if ADOT layer overhead is unacceptable):**

Create a Middy middleware that initializes the SDK per cold start:

```typescript
// packages/wallet-service/src/middlewares/tracing.ts
const tracingMiddleware = () => ({
  before: async (request) => {
    // SDK already initialized on cold start; just set span attributes
    const span = trace.getActiveSpan();
    if (span) {
      span.setAttribute('lambda.function', request.context.functionName);
      span.setAttribute('http.route', request.event.path);
    }
  },
});
```

**Cold start overhead:** ADOT layer adds 200-800ms to cold starts. For our use case this is acceptable since most endpoints already have cold starts in that range, and warm invocations add <10ms overhead.

## Collector deployment

**For the daemon (K8s):** Deploy the OTel Collector as a sidecar container in the same pod, or as a DaemonSet. The collector receives spans via OTLP and forwards to the backend.

**For Lambdas:** The ADOT layer includes an embedded collector. Configuration is via a `collector.yaml` file bundled with the deployment.

## Metrics (phase 2)

OTel metrics are experimental in JS SDK 2.0 but usable. Key metrics to add:

- `db.query.duration` — histogram of MySQL query durations (auto-instrumented)
- `http.server.request.duration` — histogram of API request durations
- `daemon.event.processing.duration` — custom histogram for event processing time
- `daemon.sync.lag` — gauge showing how far behind the daemon is from the fullnode tip

These can be exported to Prometheus (pull) or OTLP (push) depending on the backend choice.

## Sampling strategy

For production, use a **tail-based sampling** strategy at the collector level:

- Always keep traces with errors or high latency (>2s).
- Sample 10% of successful, fast traces.
- Always keep traces from critical paths (void handling, reorgs).

This keeps storage costs manageable while ensuring we never miss problematic traces.

## Rollout plan

1. **Phase 1 — Daemon tracing (2-3 days):** Add `tracing.ts`, deploy with OTLP exporter pointing to a test collector. Validate spans appear in the backend. No code changes to handlers.
2. **Phase 2 — Lambda tracing (2-3 days):** Add ADOT layer to staging Lambdas. Validate traces for API calls. Measure cold start impact.
3. **Phase 3 — Custom spans (1 week):** Add manual spans to critical paths: `handleVoidedTx`, `handleVertexAccepted`, balance validation, reorg handling.
4. **Phase 4 — Metrics and dashboards (1 week):** Add custom metrics, build dashboards for p50/p95/p99 latencies, set up alerting on SLO breaches.
5. **Phase 5 — Production rollout:** Enable in production with sampling. Monitor overhead.

Each phase can be rolled back independently by removing the layer/import.

# Drawbacks
[drawbacks]: #drawbacks

- **Cold start overhead for Lambdas.** ADOT adds 200-800ms to cold starts. For infrequently-called functions this could be noticeable. Mitigated by provisioned concurrency for critical endpoints.
- **Dependency footprint.** `@opentelemetry/auto-instrumentations-node` pulls in many sub-packages. Increases bundle size by ~5-10MB for the daemon (acceptable for K8s), more concerning for Lambda deployment packages.
- **Operational complexity.** Running a collector (or relying on ADOT) adds another component to monitor. If the collector is down, traces are lost (though the application continues to function).
- **Learning curve.** Team needs to learn OTel concepts (spans, traces, context propagation, sampling). The API is stable but the ecosystem moves fast.
- **Storage costs.** Traces generate significant data volume. Without sampling, costs can grow quickly. Tail-based sampling mitigates this but adds collector complexity.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

**Why OpenTelemetry?**

- **CNCF graduated** (same level as Kubernetes, Prometheus) — not going away.
- **Vendor-neutral** — we can switch backends (X-Ray → Tempo → Jaeger) without changing application code.
- **Auto-instrumentation** — mysql2, ioredis, aws-sdk, http are all covered out of the box. This means 80% of the value (DB query durations, HTTP call latencies) requires zero code changes.
- **Industry standard** — OTel has become the de facto standard for observability. AWS, GCP, Azure, Datadog, Grafana, and others all support it natively.

**Alternatives considered:**

1. **AWS X-Ray SDK directly** — Simpler setup for Lambda, but locks us into AWS. No standard API; switching to another backend would require rewriting instrumentation. X-Ray can still be used as a *backend* with OTel as the *instrumentation layer*.

2. **Datadog / New Relic / Honeycomb (commercial APM)** — Excellent developer experience but expensive at scale ($15-25/host/month + trace ingestion fees). Vendor lock-in. OTel can export to these if we decide to use them later.

3. **Manual logging with structured JSON** — Cheapest option, but no distributed tracing, no automatic span correlation, no flame graphs. We would need to manually add timing to every function we care about and build our own dashboards. Does not scale.

4. **Prometheus + custom metrics only** — Good for aggregate metrics but no per-request traces. Cannot answer "why was *this specific request* slow?" Only answers "what is the average latency?" Complementary to OTel, not a replacement.

**Impact of not doing this:**

We continue operating blind. Incident response remains slow (hours of log-grepping). Performance regressions ship undetected. Capacity planning remains guesswork.

# Prior art
[prior-art]: #prior-art

- **AWS Lambda + OTel:** AWS officially supports OTel via ADOT Lambda layers. AWS documentation recommends OTel over the X-Ray SDK for new projects. Many production Lambda-based services use this pattern successfully.

- **Grafana Tempo + OTel:** Grafana Labs designed Tempo specifically as an OTel-native trace backend. It uses object storage (S3) for cost-effective trace storage and integrates with Grafana dashboards for visualization. This is a proven pattern in the Kubernetes ecosystem.

- **Node.js OTel in production:** Companies like Stripe, GitHub, and Shopify use OTel for Node.js services in production. The JS SDK 2.0 (released March 2025) stabilized the tracing API and improved performance significantly.

- **OpenTelemetry Collector as sidecar:** The collector-as-sidecar pattern is standard in Kubernetes. It decouples the application from the backend, allows local buffering, and enables tail-based sampling without application changes.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Which trace backend?** The main candidates are:
  - **AWS X-Ray** — simplest for Lambda (built-in ADOT support), but limited query capabilities and no native Grafana integration.
  - **Grafana Tempo** — cost-effective (uses S3), excellent Grafana integration, but requires running the Tempo service (or using Grafana Cloud).
  - **Jaeger v2** — mature, CNCF graduated, supports OTLP natively, but requires hosting.
  - **SigNoz** — open-source all-in-one (traces + metrics + logs), based on ClickHouse, but less mature.
  This decision depends on our existing infrastructure (do we already run Grafana? are we willing to host another service?).

- **ADOT layer cold start impact in our specific Lambdas.** The 200-800ms range is from AWS documentation. We need to benchmark with our actual deployment packages to get exact numbers.

- **Sampling rates for production.** 10% baseline sampling is a starting point. The right rate depends on our trace volume and storage budget, which we will learn after the staging rollout.

- **Trace context propagation between daemon and Lambdas.** The daemon calls Lambdas via AWS SDK (SQS, direct invoke). OTel auto-instruments aws-sdk, but we need to verify that trace context propagates correctly through SQS messages so that daemon traces connect to Lambda traces.

- **Should we replace Winston with OTel Logs?** OTel Logs in the JS SDK are still experimental. For now, keep Winston and correlate logs with traces via trace IDs injected into log lines. Revisit when OTel Logs stabilize.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Continuous profiling.** OTel is adding profiling signals. Once stable, we could attach CPU/memory profiles to specific traces to debug performance issues at the code level.

- **SLO-based alerting.** With latency histograms, we can define SLOs (e.g., "99% of /wallet/balance requests complete in <500ms") and alert on burn rate rather than raw thresholds.

- **Trace-driven testing.** Record production traces and replay them in staging to validate that code changes do not introduce regressions.

- **Business metrics from spans.** Extract custom metrics from trace data: transactions processed per second, average reorg depth, void handling throughput. This avoids duplicating metric collection in application code.

- **Common package instrumentation.** Create a shared `@wallet-service/tracing` package in the monorepo that provides pre-configured OTel setup, common span attributes, and helper functions. This ensures consistent instrumentation across daemon and wallet-service.

- **Extend to event-downloader.** The new `event-downloader` package could also benefit from tracing, especially for understanding fullnode event processing latencies.
