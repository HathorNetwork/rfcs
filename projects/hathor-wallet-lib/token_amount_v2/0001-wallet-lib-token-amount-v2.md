- Feature Name: wallet-lib-token-amount-v2
- Start Date: 2026-06-10
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

Hathor is introducing **Token Amount V2**: token output values move from the
legacy 2-decimal interpretation (V1, where `123` means `1.23`) to an 18-decimal
interpretation (V2, where `1000000000000000000` means `1` whole token). The
canonical V1↔V2 logic lives in the Rust crate `htr-lib`, exposed to JavaScript
through the `@hathor/htr-lib` Node.js binding (see [bindind design](./0000-htr-lib-nodejs-bindings.md)).

This RFC compares **two ways the wallet-lib could adopt V2**, and estimates the
effort of each:

- **Approach A: Normalize at the boundary.** Keep the wallet-lib's internal
  amount type as the native `bigint` it already uses; convert any incoming V1
  amount to its V2 form at the edges (network, storage, user input) and work in
  V2 everywhere inside. The binding is used only for conversion/validation.
- **Approach B: `TokenAmount` everywhere.** Replace the internal `bigint` amount
  type with the binding's `TokenAmount`/`TokenBalance` wrapper classes across the
  whole library, so all arithmetic flows through the Rust implementation.

A fact that frames both options: because the wallet-lib **already uses `bigint`**
for every amount (migrated in Dec 2024), Approach A is a focused boundary change,
while Approach B is a library-wide rewrite plus a breaking major release for
every downstream consumer. Approach A is not entirely free of downstream impact,
though: adopting V2 as the default changes how integer amount values are
*interpreted* across the public API (integer `1` shifts from `0.01` to
`0.000000000000000001`), so consumers still need minimal changes to their display
and validation logic, even though the `bigint` type itself is unchanged. This RFC
lays out the benefits, drawbacks, and effort of each approach so the team can
decide; it does not advocate for either.

# Motivation
[motivation]: #motivation

The wallet-lib must correctly handle V2 amounts: parse them, validate them,
display them, compute balances with them, and build transactions that spend
them, while remaining compatible with the V1 amounts already on-chain. The open
question is *not whether* to support V2, but *how the library should represent an
amount internally*, because that choice determines:

- how much code changes and how risky the change is,
- whether downstream consumers (wallet-headless, wallet-mobile, wallet-service,
  the web wallet) face a breaking upgrade,
- runtime performance of hot paths like balance computation,
- and how tightly the wallet stays aligned with the full node's canonical logic.

This document lays the two options side by side with explicit trade-offs and
effort estimates so the team can make an informed call.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The starting point: the wallet-lib already speaks `bigint`

A critical fact frames this entire decision. The wallet-lib **already migrated
every token amount from `number` to `bigint`** (the `feat: implement bigint
support` change, Dec 2024). There is a single canonical type alias:

```typescript
// src/types.ts
export type OutputValueType = bigint;
```

used in ~225 places across ~29 source files. Amounts are `bigint` end to end:
in transaction outputs, balances, UTXO selection, serialization, and every
public method signature. Decimal places are *already* runtime-configurable
(`Storage.getDecimalPlaces()` reads `decimal_places` from the full node's version
API), and the display helper `numberUtils.prettyValue(value, decimalPlaces)` is
already decimal-agnostic.

So this is **not** a "replace numbers with big integers" project; that work is
done. V2 is about **decimal semantics, value range, conversion at boundaries,
and display**, not the numeric type.

This is what makes the two approaches so different in cost.

## Approach A: Normalize at the boundary

Internally, the wallet-lib continues to use native `bigint`, but every amount it
holds is **always a V2-normalized value**. The library establishes a rule:

> Inside the wallet-lib, an amount is always V2. V1 only exists at the edges.

Whenever a V1 amount enters (parsing an old transaction from the network, an
API response, or stored data), it is immediately converted to V2 using the
`@hathor/htr-lib` binding. Whenever an amount leaves toward something that still
expects V1, it is converted back (and the binding tells us if that conversion
would truncate). Everything between the edges is untouched: `balance += value`,
`change = inputs - outputs - fee`, `value > 0n` all stay exactly as they are,
because they were already correct `bigint` operations.

```typescript
// At the network/storage boundary: convert V1 in, work in V2:
const v2 = TokenAmount.fromV1(rawV1Value).normalized(); // bigint, V2-normalized
// ... everything downstream is ordinary bigint arithmetic, unchanged ...
balance += v2;
```

The binding is the **source of truth for the conversion math only**. The rest of
the library's "inner workings are kept the same."

## Approach B: `TokenAmount` everywhere

The wallet-lib replaces `OutputValueType = bigint` with the binding's wrapper
classes: `TokenAmount` for unsigned amounts, `TokenBalance` for signed balances.
Every amount the library handles becomes a wrapper object, and every operation on
it goes through the Rust implementation:

```typescript
// Every arithmetic site changes shape:
balance = balance.add(value);          // was: balance += value
const change = inputs.sub(outputs).sub(fee);  // was: inputs - outputs - fee
if (amount.gt(TokenAmount.zero())) {}  // was: if (amount > 0n)
if (a.eq(b)) {}                        // was: if (a === b)
```

The appeal is **type safety and zero drift**: a V1 amount added to a V2 amount is
reconciled by Rust, and any future change to `htr-lib`'s semantics propagates the
moment the binding is upgraded. The cost is that **JavaScript has no operator
overloading** (see [bindind design](./0000-htr-lib-nodejs-bindings.md)), so every `+`, `-`, and `===` on an amount must
become a method call, across the entire library, its public API, its tests, and
every consumer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Work shared by both approaches (the baseline)

Regardless of which approach is chosen, V2 support requires the following. These
are listed first so the comparison that follows isolates only the *difference*
between A and B.

| Shared work item            | What it involves                                                                                                                                                                                          | Key files                                 |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| Integrate `@hathor/htr-lib` | Add the native binding + ensure the WASM fallback actually loads in React Native (wallet-mobile) and the browser (electron wallets, browser, ), not just Node. **Highest-risk item.**                     | new dependency; loader config             |
| Per-token decimals          | `getDecimalPlaces()` is per-*network* today; V2 needs decimals tracked per *token* (V1 vs V2 tokens coexist). Token-metadata plumbing.                                                                    | `src/storage/storage.ts`, token metadata  |
| Value range / wire format   | `MAX_OUTPUT_VALUE = 2^63` caps a single output at ~9.2 whole tokens at 18 decimals (a hard blocker). Encode/decode of V2 values. *Gated on the core's encoding decision (see new-decimal-precision RFC).* | `src/constants.ts`, `src/utils/buffer.ts` |
| Deposit / melt precision    | `getDepositAmount`/`getWithdrawAmount` round-trip through `Number()` with a float multiply, unsafe at 18 decimals; must move to integer math.                                                             | `src/utils/tokens.ts`                     |
| Display & input parsing     | Formatting (`prettyValue`, already param-driven) and parsing user-entered decimal strings into V2 base units.                                                                                             | `src/utils/numbers.ts`                    |

**Estimated shared baseline: ~16–24 dev-days**, with the binding's
cross-platform integration carrying most of the schedule risk.

## Approach A: additional work

On top of the baseline, Approach A adds only conversion and validation at the
boundaries; internal arithmetic is untouched.

| A-specific work item   | What it involves                                                                                                                                               |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Boundary normalization     | Convert V1→V2 on ingest (network parse, API schemas, stored data) and V2→V1 on egress where a V1 consumer remains; surface truncation errors from the binding.                                                                                                                  |
| Schema validation          | The ~36 `bigIntCoercibleSchema` (zod) sites validate/normalize at the JSON boundary; extend to normalize V1 values.                                                                                                                                                            |
| Public API value reinterpretation | Adopting V2 as the default changes the *meaning* of amount integers emitted/accepted by the public API (integer `1`: `0.01` → `0.000000000000000001`). The `bigint` type signature is unchanged, but consumers formatting/validating with 2 decimals must be updated. A real, if minimal, downstream change across the consuming projects. |
| Targeted tests             | Test conversion, validation, range, and display. **Existing arithmetic tests and fixtures stay valid** because the internal representation does not change.                                                                                                                    |

**Internal arithmetic (~200–250 sites), public type *signatures*, JSON
serialization, and the ~74 test files with ~2,400 hardcoded amount literals are
untouched. The *values* those public APIs carry are reinterpreted V1→V2,
however, so consuming projects need minimal display/validation updates (see the
"Public API value reinterpretation" row above).**

**Approach A delta: ~6–10 dev-days.**

## Approach B: additional work

On top of the baseline, Approach B replaces the internal type, which cascades
across the whole library and every consumer.

| B-specific work item    | What it involves                                                                                                                                                                                                                       | Magnitude (from code audit)                                   |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Rewrite arithmetic      | Every `+=`/`-=` accumulation → `.add`/`.sub`; every `===`/`!==` → `.eq`/`.ne`. (Relational `<`,`>` can survive via `Symbol.toPrimitive`, but mixing typed and untyped comparisons is a hazard.)                                        | ~135 accumulation lines across 22 files; ~53 comparison sites |
| Change public API types | `OutputValueType` is in nearly every exported interface (`IBalance`, `IUtxo`, `getBalance()`, `sendTransaction()`, …). Swapping `bigint`→`TokenAmount` is a **breaking major** for all consumers.                                      | ~225 references; ~120 exported symbols                        |
| Serialization / storage | `IStore` is implemented by consumers (mobile, desktop persist to disk). `bigint` serializes via the existing `JSONBigInt`; `TokenAmount` needs custom (de)serialization in **every** consumer store.                                   | `JSONBigInt`, all `IStore` implementations                    |
| Schema coercion         | The ~36 zod coercion sites must produce/accept `TokenAmount`, not `bigint`.                                                                                                                                                            | ~36 sites                                                     |
| Test & fixture rewrite  | Fixtures use raw `bigint` literals (`value: 1n`); assertions use `===`/`toEqual(100n)`. All must wrap/unwrap.                                                                                                                          | ~74 test files, ~2,400 literal lines                          |
| Performance validation  | Every arithmetic op now crosses the JS↔native (FFI/WASM) boundary. Summing thousands of UTXOs for a balance becomes thousands of boundary crossings, which must be benchmarked on the hot paths and on the WASM fallback (mobile/web). | balance & UTXO-selection loops                                |
| Downstream migration    | wallet-headless, wallet-mobile, wallet-service, and the web wallet each consume these types and must migrate in lockstep with the breaking release.                                                                                    | 4+ external repos                                             |

**Approach B delta: ~32–56 dev-days in the wallet-lib alone**, plus
separate migration projects in each downstream repo.

## Trade-off comparison

| Dimension                  | Approach A (Normalize at boundary)                                 | Approach B (`TokenAmount` everywhere)                    |
| -------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------- |
| **Internal amount type**   | Native `bigint` (unchanged)                                        | `TokenAmount` / `TokenBalance` wrappers                  |
| **Code churn**             | Boundaries only (~edges of ~5 files)                               | Library-wide (~200–250 arithmetic sites + types + tests) |
| **Public API impact**      | Type-compatible (still `bigint`), but amount *values* are reinterpreted V1→V2; consumers need minimal display/validation updates | **Breaking major** (every consumer must migrate)         |
| **Type safety on amounts** | Convention-enforced ("always V2"); a stray V1 value is a logic bug | Compiler-enforced; V1/V2 mixing reconciled by Rust       |
| **Drift from full node**   | Conversion math is canonical (binding); arithmetic is local        | Zero drift (all amount logic is the Rust crate)          |
| **Operator ergonomics**    | Natural `a + b`, `a > b`, `a === b` retained                       | `a.add(b)`, `a.eq(b)`; only `<`/`>` stay native          |
| **Runtime performance**    | No per-op overhead (native bigint)                                 | FFI/WASM crossing per op; matters on balance/UTXO loops  |
| **Serialization/storage**  | Unchanged (`JSONBigInt`)                                           | Custom (de)serialization in every consumer store         |
| **Binding criticality**    | Boundary-only; a binding bug has limited blast radius              | On every hot path; a binding/WASM bug affects everything |
| **Test/fixture impact**    | Mostly reused                                                      | ~74 files / ~2,400 literals rewritten                    |
| **Blast radius if wrong**  | Contained, reversible                                              | Large, hard to reverse (breaking release shipped)        |

## Effort & timeline estimation

Assumptions: figures are **dev-days** (1 engineer-week = 4 dev-days) for focused
work by an engineer familiar with the wallet-lib; calendar time assumes 1
engineer and includes
review/QA. Ranges express genuine uncertainty; the dominant schedule risk in both
is validating the binding (native + WASM fallback) across Node, React Native, and
the browser. The wire-format item is **gated on the core's encoding decision**
and may shift if that lands later.

|                             | Approach A             | Approach B                                                                                             |
| --------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------ |
| Shared baseline             | 16–24 dev-days         | 16–24 dev-days                                                                                         |
| Approach-specific delta     | 6–10 dev-days          | 32–56 dev-days                                                                                         |
| **Total (wallet-lib only)** | **~22–34 dev-days**    | **~48–80 dev-days**                                                                                    |
| Calendar (1 engineer)       | ~1.5–2 months          | ~3–5 months                                                                                            |
| Downstream consumer work    | Minimal but non-zero: amount values reinterpreted V1→V2, consumers update decimal handling | **Separate migration in 4+ repos** (headless, mobile, service, web), likely adding 1–3 months org-wide |
| Confidence                  | Medium-High            | Medium-Low (rewrite + cross-repo coordination widen the variance)                                      |

**Reading the numbers:** in the wallet-lib alone, Approach A is roughly **2–3×
cheaper** than Approach B. Approach B also carries mandatory downstream
migrations (a breaking release across 4+ repos) that Approach A does not, and a
larger estimate variance. These figures are inputs to the decision, not a verdict;
the choice also weighs the type-safety, drift, and performance dimensions in
the trade-off table above.

# Drawbacks
[drawbacks]: #drawbacks

**Approach A drawbacks**

- Type safety is by *convention* ("internally always V2"), not by the compiler.
  A V1 value that slips past a boundary is an ordinary `bigint` and will compute
  silently-wrong results. Mitigation: centralize conversion at the edges, lint
  for raw amount construction, and test boundaries hard.
- Conversion logic in the wallet-lib (deciding *where* to normalize) is local
  code, not the Rust crate, so the *placement* of conversions can drift even
  though the conversion *math* cannot.
- Not fully non-breaking at the public API. The amount integers emitted/accepted
  by public methods change meaning when V2 becomes the default (integer `1`:
  `0.01` → `0.000000000000000001`). The `bigint` type is unchanged, so it
  compiles, but every consumer that formats or validates amounts with 2 decimals
  must be updated. The change is minimal but must be coordinated across the
  consuming projects.

**Approach B drawbacks**

- A breaking major release forces every downstream consumer to migrate in
  lockstep, a large coordination cost with real release-management risk.
- Per-operation FFI/WASM overhead on hot paths (balance aggregation, UTXO
  selection) is a genuine performance concern, especially via the WASM fallback
  on mobile and web.
- Serialization becomes a cross-repo problem: every `IStore` implementation must
  (de)serialize the wrapper type instead of relying on the existing `JSONBigInt`.
- The large surface area (~200–250 sites, ~74 test files) raises the chance of a
  subtle arithmetic-translation bug surviving into a shipped breaking release.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section frames *how to weigh* the two approaches without picking one; the
decision belongs to the team reading this document.

- **The case for Approach A.** The wallet-lib already standardized on `bigint`,
  and V2 is largely a *decimal-semantics and boundary* problem rather than a
  *numeric-type* problem. Approach A addresses that with a contained, reversible
  change, while still using the canonical binding for the part that must not
  drift: the conversion math. It is the smaller, lower-variance, cheaper path,
  and its downstream impact is limited to a minimal value-reinterpretation update
  (the `bigint` type signatures do not change). Its costs are giving up
  compiler-enforced V1/V2 type safety in favor of a convention, and that the
  public-API value shift, though small, still has to be coordinated with
  consuming projects.
- **The case for Approach B.** It buys compiler-enforced V1/V2 type safety and
  zero drift on *all* amount logic, not just conversion, valuable if amount
  bugs are considered high-severity and worth strong static guarantees. The cost
  is a library-wide rewrite, a breaking release across four-plus repos, a
  per-operation performance tax on hot paths, and a cross-repo serialization
  change.
- **A hybrid path.** Start with Approach A and *optionally* introduce
  `TokenAmount` later in narrow, high-value spots (e.g. a public conversion API
  surface) without rewriting internal arithmetic. This captures part of B's
  type-safety benefit at the boundary while keeping A's internals; a door A
  leaves open that B does not. Listed as an option, not a recommendation.
- **Impact of doing nothing.** The wallet-lib cannot represent, validate, or
  display V2 amounts, blocking V2 adoption for every wallet that depends on it.

# Prior art
[prior-art]: #prior-art

- **`htr-lib-py` / full node.** The Rust crate normalizes internally to V2 and
  exposes conversions, exactly the model Approach A mirrors in JS (work in the
  normalized form, convert at the edges).
- **Ethereum JS ecosystem (`ethers.js`, `viem`).** The dominant pattern is
  native `bigint` for amounts plus helper functions (`parseUnits`/`formatUnits`)
  for decimal conversion at the boundary, i.e. Approach A's shape. Wrapper-class
  amount types (closer to Approach B) exist but did not become the ecosystem
  norm, partly because of the operator-overloading gap this RFC describes.
- **The wallet-lib's own `number`→`bigint` migration (Dec 2024).** A prior
  library-wide amount-type change; its scope is a useful real-world calibration
  for how invasive Approach B's type swap would be.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- The wire-format encoding for V2 values (and the new maximum output value) is
  decided by the core's decimal-precision work; the wallet-lib's serialization
  estimate assumes that lands and may shift if it is delayed.
- Exactly *where* per-token decimals are sourced (full node metadata vs.
  token-creation data) needs to be pinned down for the per-token decimals item.
- For Approach A: which boundaries still legitimately emit V1 (and therefore need
  V2→V1 egress conversion with truncation handling) vs. boundaries that can be
  V2-only.
- Whether the `@hathor/htr-lib` WASM fallback meets performance and bundle-size
  needs on React Native and the browser, a prerequisite for **both** approaches,
  but a far more load-bearing one for Approach B.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Incremental `TokenAmount` adoption.** Having shipped Approach A, narrow
  high-value surfaces (public conversion APIs, a typed amount in new features)
  can adopt `TokenAmount` opportunistically, without a big-bang rewrite, keeping
  the door to B's type-safety benefits open at low cost.
- **Shared validation across the stack.** As more of the stack (wallet-service,
  headless) consumes `@hathor/htr-lib`, V1/V2 validation and conversion converge
  on one implementation, reducing the surface for cross-service inconsistencies.
- **Web3-style helpers.** A thin `parseUnits`/`formatUnits`-style layer over the
  binding would give wallet developers a familiar, ergonomic API for V2 amounts
  regardless of the internal representation chosen here.
