- Feature Name: htr-lib-nodejs-bindings
- Start Date: 2026-06-10
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

Expose the pure-Rust `htr-lib` domain types (`TokenAmount`, `TokenBalance`,
`TokenAmountVersion`) to TypeScript/JavaScript through a
[napi-rs](https://napi.rs) binding, published to npm as `@hathor/htr-lib`. The
binding mirrors the role `htr-lib-py` plays for Python: it is the single,
canonical implementation of token-amount arithmetic and the V1↔V2 normalization
logic, shared across languages. A WebAssembly artifact provides a universal
fallback for environments without a prebuilt native binary (browsers, Deno, Bun,
edge runtimes, unsupported architectures). An initial implementation is under
development in [hathor-core PR #1725](https://github.com/HathorNetwork/hathor-core/pull/1725).

# Motivation
[motivation]: #motivation

Hathor is introducing a new "Token Amount V2" representation: token output
values move from the legacy 2-decimal interpretation (where `123` means `1.23`)
to an 18-decimal representation (where `1000000000000000000` means `1` unit).
This affects every place an amount is serialized, normalized, compared, or
arithmetically combined, across the full node, the wallet libraries, and any
external integration.

If each language re-implements the normalization factor, the truncation rules,
the overflow guards, and the V1/V2 conversion semantics independently, the
implementations *will* drift. A subtle disagreement about, say, when a V2→V1
conversion is allowed to truncate is a consensus-relevant bug. The Rust crate
`htr-lib` is the agreed source of truth; `htr-lib-py` already binds it for
Python. This RFC defines the equivalent binding for the Node.js/TypeScript
ecosystem so that the wallet-lib and other JS consumers compute amounts with
exactly the same code the full node uses.

The expected outcome is a published npm package, `@hathor/htr-lib`, whose
TypeScript types are generated from the Rust source, so the JS surface cannot
silently fall out of sync with the Rust domain logic.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`@hathor/htr-lib` exposes two value types and one enum:

- `TokenAmount`: an **unsigned** token amount. Internally normalized to the V2
  (18-decimal) representation regardless of which version produced it.
- `TokenBalance`: a **signed** balance (can be negative), used for balance
  accounting where inputs and outputs net out.
- `TokenAmountVersion`: an enum `{ V1 = 1, V2 = 2 }` identifying the
  interpretation an amount was created with.

The most important thing to internalize is that **JavaScript has no operator
overloading**. In Python, `htr-lib-py` can expose `a + b`, `a == b`, and
`a < b` directly via dunder methods (`__add__`, `__eq__`, `__lt__`). JavaScript
cannot do the equivalent for two custom objects. So arithmetic and equality are
exposed as **named methods**, and only the four relational operators get native
"sugar".

## Numbers cross the boundary as `bigint`

Rust's `BigUint`/`BigInt` map to JavaScript's native `bigint`. There is no
precision loss and no `number`/float involved:

```typescript
import { TokenAmount } from '@hathor/htr-lib';

const oneTokenV2 = TokenAmount.fromV2(1_000_000_000_000_000_000n); // 1 unit, 18 decimals
const oneTokenV1 = TokenAmount.fromV1(100n);                       // "1.00" in legacy form

oneTokenV1.normalized();  // => 1_000_000_000_000_000_000n  (always V2-normalized)
oneTokenV1.raw();         // => 100n                        (the value as stored for its version)
oneTokenV1.eq(oneTokenV2); // => true  (same normalized value, different versions)
```

## Arithmetic and equality are methods

```typescript
const a = TokenAmount.fromV2(10n);
const b = TokenAmount.fromV2(3n);

const c = a.add(b);   // TokenAmount(13)   (NOT a + b)
const d = a.sub(b);   // TokenAmount(7)
a.eq(b);              // false             (NOT a === b)
a.ne(b);              // true
```

`a + b` and `a === b` are deliberately **not** supported on `TokenAmount`. `+`
would coerce both operands to raw `bigint` and silently drop the wrapper type;
`===` between two objects compares reference identity and would always be
`false`. Using the methods is the only correct path.

## Relational operators work natively

The four relational operators (`<`, `>`, `<=`, `>=`) *do* work, because
JavaScript coerces operands through `[Symbol.toPrimitive]` before comparing:

```typescript
const a = TokenAmount.fromV2(5n);
const b = TokenAmount.fromV2(8n);

a < b;        // true   (native, via Symbol.toPrimitive)
a.lt(b);      // true   (explicit method, equivalent)
a.compare(b); // -1     (-1 | 0 | 1)
```

Both forms exist. The methods (`lt`/`le`/`gt`/`ge`/`compare`) are the
recommended, type-safe form; the native operators are ergonomic sugar with one
caveat (see [cross-type safety](#semantics--error-mapping)).

## Versions and conversions

```typescript
import { TokenAmount, TokenAmountVersion } from '@hathor/htr-lib';

const amount = TokenAmount.fromV1(150n);   // "1.50" legacy
amount.isV1();        // true
amount.isV2();        // false

const asV2 = amount.toV2();                 // always succeeds (V1 ⊂ V2)
const backToV1 = amount.toV1();             // succeeds (150 fits the 2-decimal grid)

// A V2 value with sub-V1 precision cannot become V1 without truncating:
const fine = TokenAmount.fromV2(1_500_000_000_000_000_001n);
fine.toV1();        // throws Error: "cannot denormalize value, would truncate (...)"
fine.maybeToV1();   // returns null instead of throwing
```

## Balances are signed

`TokenBalance` is the type to reach for when accounting can go negative
(e.g. a wallet's net position while summing inputs and outputs):

```typescript
import { TokenBalance } from '@hathor/htr-lib';

const bal = new TokenBalance(5n);
const neg = bal.sub(new TokenBalance(8n));  // TokenBalance(-3)
neg.asBool();        // true   (raw != 0)
neg.isZero();        // false
neg.toAmount();      // throws Error: "cannot convert negative TokenBalance to TokenAmount (...)"
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Toolchain

- **napi-rs v3** (chosen over v2 specifically for its WASM target support) plus
  `@napi-rs/cli`.
- The CLI **auto-generates** `index.js` and `index.d.ts` from `#[napi]`
  annotations. TypeScript types are generated, never hand-written, which is what
  keeps the JS surface locked to the Rust source.
- Naming convention: Rust `snake_case` → JS `camelCase` (napi-rs default).
- Numbers: `BigUint`/`BigInt` ↔ JS `bigint`.

## Crate and file layout

A new workspace crate `crates/htr-lib-napi`, compiled as a `cdylib` native addon
and (via emnapi) to a `wasm32-wasi` artifact. The layout mirrors `htr-lib-py`:

```
crates/htr-lib-napi/
  Cargo.toml            # cdylib, napi + napi-derive deps, build-dep napi-build
  build.rs              # napi_build::setup()
  package.json          # name @hathor/htr-lib, @napi-rs/cli scripts, targets, optionalDependencies
  src/
    lib.rs              # module wiring
    token_amount.rs     # NapiTokenAmount
    token_balance.rs    # NapiTokenBalance
  __tests__/            # ava integration tests
  npm/                  # generated per-target packages (binaries added at publish time)
    linux-x64-gnu/  linux-arm64-gnu/  linux-x64-musl/  linux-arm64-musl/
    darwin-x64/  darwin-arm64/  win32-x64-msvc/  wasm32-wasi/
  index.js              # generated by @napi-rs/cli
  index.d.ts            # generated by @napi-rs/cli
  README.md             # usage + manual build/publish steps
```

`index.js` / `index.d.ts` are generated artifacts. Whether they are committed or
git-ignored follows the napi-rs default for the chosen template and the
surrounding repo convention; this RFC does not mandate one.

## TypeScript API surface

```typescript
export const enum TokenAmountVersion { V1 = 1, V2 = 2 }

export class TokenAmount {
  static setNormalizationFactor(v1DecimalPlaces: number, v2DecimalPlaces: number): void
  static getNormalizationFactor(): bigint
  static fromV1(amount: bigint): TokenAmount
  static fromV2(amount: bigint): TokenAmount
  static fromVersion(amount: bigint, version: TokenAmountVersion): TokenAmount
  static zero(): TokenAmount

  isV1(): boolean
  isV2(): boolean
  normalized(): bigint
  raw(): bigint
  asBool(): boolean             // raw != 0  (JS truthiness can't be overridden)
  isZero(): boolean             // raw == 0  (convenience inverse of asBool)
  toBalance(): TokenBalance
  toV1(): TokenAmount           // throws Error if it would truncate
  maybeToV1(): TokenAmount | null
  toV2(): TokenAmount
  toVersion(version: TokenAmountVersion): TokenAmount  // throws Error if truncate

  add(other: TokenAmount): TokenAmount
  sub(other: TokenAmount): TokenAmount

  eq(other: TokenAmount): boolean
  ne(other: TokenAmount): boolean
  lt(other: TokenAmount): boolean
  le(other: TokenAmount): boolean
  gt(other: TokenAmount): boolean
  ge(other: TokenAmount): boolean
  compare(other: TokenAmount): number          // -1 | 0 | 1

  toString(): string                           // mirrors Python __repr__ (Rust Debug)
  [Symbol.toPrimitive](hint: string): bigint   // returns normalized; enables native <, >, <=, >=
}

export class TokenBalance {
  constructor(balance?: bigint)                // default 0n
  raw(): bigint
  asBool(): boolean                            // raw != 0
  isZero(): boolean                            // raw == 0
  toBalance(): TokenBalance                    // identity
  toAmount(): TokenAmount                       // throws Error if negative

  add(other: TokenBalance): TokenBalance
  sub(other: TokenBalance): TokenBalance
  neg(): TokenBalance

  eq(other: TokenBalance): boolean
  ne(other: TokenBalance): boolean
  lt(other: TokenBalance): boolean
  le(other: TokenBalance): boolean
  gt(other: TokenBalance): boolean
  ge(other: TokenBalance): boolean
  compare(other: TokenBalance): number         // -1 | 0 | 1

  toString(): string
  [Symbol.toPrimitive](hint: string): bigint   // returns raw; enables native <, >, <=, >=
}
```

## Design decisions

### Methods + relational sugar

The table below is the core constraint that drives the whole API shape:

| Operator          | Possible on two custom objects? | Mechanism / reason                                                          |
| ----------------- | ------------------------------- | --------------------------------------------------------------------------- |
| `<` `>` `<=` `>=` | Yes                             | Relational operators coerce operands via `[Symbol.toPrimitive]`.            |
| `+` `-`           | Harmful                         | Coerce too, but yield a raw `bigint`, dropping the wrapper type. Not used.  |
| `==` `!=`         | No (object vs object)           | No coercion between two objects; falls back to reference identity.          |
| `===` `!==`       | No                              | Strict equality never coerces.                                             |

Hence: arithmetic (`add`/`sub`/`neg`) and equality (`eq`/`ne`) are methods; the
four relational operators are *additionally* supported natively via
`[Symbol.toPrimitive]`.

### `TokenAmountVersion` as an enum

Exported as a napi enum `{ V1 = 1, V2 = 2 }`, not a raw number. This is
type-safe and eliminates the "unknown version" runtime path Python has to carry.

### Comparison value source

- `TokenAmount` orders and compares on `normalized()`, so its
  `[Symbol.toPrimitive]` returns the normalized value, keeping native `<`/`>`
  consistent with `compare()`/`lt()`.
- `TokenBalance` orders and compares on `raw` (signed), so its
  `[Symbol.toPrimitive]` returns `raw`.

## Semantics & error mapping

- **Cross-type safety:** the `eq`/`ne`/`lt`/… methods take a typed
  `TokenAmount`/`TokenBalance`; napi-rs runtime-validates the argument and throws
  a `TypeError` on a foreign type, matching Python `__richcmp__`'s guard *on the
  methods*. The native `<`/`>` sugar via `Symbol.toPrimitive` does **not** carry
  that guard; coercion produces two `bigint`s and JS compares them with no type
  check. This is a known, accepted tradeoff of enabling native relational ops.
- **Fallible conversions:** `toV1`, `toVersion` (to V1), and `toAmount` return
  `Option::None` in Rust on failure (truncation / negative); the binding throws
  an `Error` carrying the same descriptive message text Python uses (e.g.
  `"cannot denormalize value, would truncate (...)"`,
  `"cannot convert negative TokenBalance to TokenAmount (...)"`). `maybeToV1`
  returns `null` instead of throwing.
- **Arithmetic underflow:** `TokenAmount.sub` underflow panics in Rust (a
  `BigUint` cannot be negative); napi-rs catches the panic and surfaces it as a
  thrown JS error. `TokenBalance` is signed and does not underflow; use it when
  a negative result is legitimate.
- **Global factor:** `setNormalizationFactor` is idempotent for an equal factor
  and panics (→ thrown error) on a conflicting re-set, unchanged from `htr-lib`.
  The factor is backed by a process-wide `OnceLock`.

## Packaging

For external consumers across multiple architectures:

- **Main package** `@hathor/htr-lib`: JS loader + `index.d.ts`, listing each
  platform package as an `optionalDependency`. The loader prefers a matching
  native binary and falls back to the `wasm32-wasi` package when none matches.
- **Per-platform packages** under `npm/`, one `.node` (or WASM) binary each, with
  `os`/`cpu`/`libc` fields so npm installs only the matching one:
  `linux-x64-gnu`, `linux-arm64-gnu`, `linux-x64-musl`, `linux-arm64-musl`,
  `darwin-x64`, `darwin-arm64`, `win32-x64-msvc`, and `wasm32-wasi` (the
  universal fallback: browsers, Deno, Bun, edge runtimes, unsupported arches).

### Publishing

**No CI / automated publishing is added in this PR.** The package structure is
set up to be publishable, and the README documents the manual steps: building
each target with `napi build --release --target <triple>`, populating the `npm/`
directories, and publishing with `npm publish` (and `napi prepublish` as
appropriate). Building all targets locally requires the relevant
cross-compilation toolchains; the README notes this. Publishing is done manually
for now.

## Testing

- **ava integration tests** (`__tests__/`) exercise the real JS-facing surface:
  `bigint` round-trips, every method, native `<`/`>`/`<=`/`>=` via
  `Symbol.toPrimitive`, thrown errors (truncation, negative `toAmount`,
  cross-type comparison), cross-version equality/ordering, and the `toV1`
  truncation cases. ava is the test runner napi-rs scaffolds by default.
- **Rust `#[cfg(test)]` unit tests** cover only pure helper logic not naturally
  reachable from JS (e.g. `TokenAmountVersion` mapping, error-message
  construction).
- **Global-factor caveat:** unlike Rust's nextest (process-per-test isolation),
  the ava test process shares one `OnceLock`-backed normalization factor. Tests
  set a single agreed factor once for the process rather than per-test.

# Drawbacks
[drawbacks]: #drawbacks

- **No ergonomic arithmetic.** `c = a.add(b)` instead of `c = a + b` is verbose
  and unfamiliar. Consumers who want operator ergonomics must use native
  `bigint` and lose the wrapper type; this is the central tension the companion
  RFC (`0001`) analyzes for the wallet-lib.
- **Two ways to compare.** Both `a.lt(b)` and `a < b` work, but only the method
  form is type-guarded. Mixing the two invites subtle bugs where a native
  comparison silently coerces a foreign object.
- **A native dependency.** Shipping prebuilt `.node` binaries for eight targets
  plus a WASM fallback is operational overhead (cross-compilation toolchains,
  per-platform packages, manual publishing for now). A pure-JS implementation
  would avoid this, at the cost of the cross-language drift this RFC exists to
  prevent.
- **Process-global normalization factor.** The `OnceLock` factor is convenient
  but makes the binding stateful in a way that complicates testing and forbids
  two different factors in one process.
- **`Symbol.toPrimitive` foot-gun.** Native relational operators bypass the
  cross-type guard. A future maintainer may not realize `tokenAmount < someBalance`
  coerces both sides and compares unrelated magnitudes without complaint.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- **Why a Rust binding at all?** The normalization factor, truncation rules, and
  conversion semantics are consensus-relevant. A second hand-written JS
  implementation would drift from the full node. Binding the canonical Rust crate
  makes drift structurally impossible: the TS types are *generated* from the Rust
  source.
- **Why napi-rs v3 over v2?** v3 adds first-class WASM target support, which is
  required for the universal fallback (browsers, Deno, Bun, edge). v2 would force
  a separate WASM toolchain.
- **Why methods + relational sugar, not pure methods?** Native `<`/`>` are free
  (JS coerces via `Symbol.toPrimitive` anyway) and dramatically improve
  readability for the common case of ordering checks. The accepted cost is that
  the native form skips the type guard.
- **Why not expose `+`/`-` via `Symbol.toPrimitive`?** They would coerce to raw
  `bigint` and silently return the wrong type (a `bigint`, not a `TokenAmount`),
  hiding bugs. Comparisons return `boolean` either way, so there is no type to
  lose, which is exactly why relational sugar is safe but arithmetic sugar is
  not.
- **Impact of not doing this:** the wallet-lib (and every JS integration) must
  re-implement V1/V2 normalization by hand, accepting permanent drift risk
  against the full node.

# Prior art
[prior-art]: #prior-art

- **`htr-lib-py`** is the direct sibling. It binds the same `htr-lib` crate via
  PyO3 and uses Python dunder methods for full operator overloading. This RFC is
  the JS analogue; the API is intentionally a 1:1 mapping where the language
  allows, and a documented divergence (methods instead of operators) where it
  does not.
- **napi-rs** is the established standard for Rust↔Node native modules; major
  projects (SWC, Prisma's query engine, Parcel's transformer) ship multi-target
  prebuilt binaries with a WASM fallback in exactly this shape.
- **Ethereum's `ethers.js`/`viem`** model amounts as native `bigint` with helper
  functions (`parseUnits`/`formatUnits`) rather than a wrapper class, evidence
  that the JS ecosystem leans toward `bigint`-plus-helpers over wrapper types,
  which is relevant to the companion RFC's "keep bigint" option.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should `index.js` / `index.d.ts` be committed or git-ignored? Deferred to the
  napi-rs template default and repo convention.
- The exact normalization factor (V1 = 2 decimals, V2 = 18 decimals → factor
  `10^16`) is set by the V2 decimal-precision decision tracked separately; this
  binding only consumes it via `setNormalizationFactor`.
- Whether the wallet-lib adopts `TokenAmount` as a first-class type or keeps
  native `bigint` and uses this binding only at boundaries, the subject of
  RFC `0001`.

# Future possibilities
[future-possibilities]: #future-possibilities

- **CI / automated multi-target publishing** via GitHub Actions (build matrix per
  triple, `napi prepublish`), replacing the manual process documented in the
  README.
- **Additional domain types.** As `htr-lib` grows (e.g. richer
  transaction-amount helpers, fee math), the same binding pattern extends to new
  `#[napi]` types with no change to the packaging or publishing story.
- **Browser-first ergonomics.** Now that a WASM artifact exists, a thin
  higher-level wrapper could expose `parseUnits`/`formatUnits`-style helpers
  familiar to web3 developers, layered on top of the generated bindings.
