- Feature Name: ledger-app-token-amount-v2-impact
- Start Date: 2026-06-10
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

This document assesses the impact of Token Amount V2 (18-decimal amounts and a
new variable-length output value encoding, see the
[wallet-lib design](./0001-wallet-lib-token-amount-v2.md)) on the
[Hathor Ledger app](https://github.com/HathorNetwork/hathor-ledger-app) and on
its hardcoded client in the desktop wallet
([`public/ledger.js`](https://github.com/HathorNetwork/hathor-wallet/blob/master/public/ledger.js)).

The short version: **the Ledger app cannot handle V2 amounts in any form
today**, and, more importantly, **an un-updated app does not cleanly reject a V2
transaction; it would mis-parse it**, which breaks the "what you see is what you
sign" guarantee. Shipping V2 therefore requires either updating the app (which
is outdated on several other fronts and still targets the deprecated Nano S) or
explicitly fencing Ledger users out of V2 on the host side.

# Current state of the app
[current-state]: #current-state

| Aspect | State |
| --- | --- |
| Version / activity | v1.1.1 (2024-03); repo effectively dormant since mid-2024 |
| Devices | Nano S, Nano X, Nano S Plus. No Stax / Flex support |
| UI stack | Legacy BAGL/`UX_FLOW` only; no NBGL port (NBGL is required for Stax/Flex) |
| SDK style | Old BOLOS conventions (`THROW`/`BEGIN_TRY`); not the modern standard-app layout |
| Feature gaps | No authority outputs (mint/melt), no nano contract calls, no timelocks, P2PKH-only scripts |

Two facts shape every option below:

1. **Nano S is deprecated by Ledger** and current SDK majors no longer support
   it. Any modernization implicitly drops Nano S, while a minimal patch on the
   old SDK keeps it but deepens the technical debt.
2. **The app client is hardcoded in the desktop wallet.** `public/ledger.js` is
   a thin APDU pipe; the actual transaction bytes the device parses come from
   the wallet-lib (`tx.getDataToSign()`), and the protocol/version gates live in
   the desktop wallet's constants. Host-side changes span both repos plus the
   wallet-lib.

# What V2 breaks on the device
[break-points]: #break-points

Exact break points in the app source:

| # | Location | Problem |
| --- | --- | --- |
| 1 | `src/transaction/deserialize.c:41-64` | Output value decoder implements only the V1 4/8-byte scheme (high bit selects width, 8-byte values stored negated). A length-prefix V2 encoding requires a full rewrite of this parser. |
| 2 | `src/transaction/types.h:33` | Values are held in a `uint64_t`. Wire values above 2^63 cannot be represented; V2 values beyond that need a byte-array representation and bignum-style decimal rendering (BOLOS has no native 128-bit integer). |
| 3 | `src/common/format.c:147-176` | Amount formatting hardcodes 2 decimals (`value / 100`, `value % 100`) and has no output-length parameter. An SDK helper that takes a decimals argument exists in the repo but is dead code. |
| 4 | `src/ui/display.c:21` | The display buffer is `char g_amount[30]`. An 18-decimal amount needs at least 21 characters after the decimal point alone; the buffer is already borderline for extreme V1 values. |
| 5 | `src/handler/sign_tx.c:167-172` | The sign-tx protocol version byte accepts only the legacy values and `1`; anything else is rejected. V2 needs a new protocol version and a new parse path behind it. |
| 6 | `src/handler/sign_tx.c:190-202` | **The transaction version field is read but never validated.** An old app receiving a V2 transaction does not refuse it; it mis-parses the bytes with V1 rules. Whether this yields garbage (rejected) or a plausible-looking wrong amount (signed!) depends on the bytes. This is the safety-critical gap. |
| 7 | `src/token/types.h:14-21` | Token metadata (uid, symbol, name, version byte) carries no decimals. The version byte is the designed extension hook and is included in the token signature MAC, so introducing a version-2 token record with decimals is backward-safe but requires re-signing registered tokens. |
| 8 | Tests and fuzzing | `fuzzing/fuzz_tx_parser.cc`, `unit-tests/test_format.c`, and the Python test client all encode V1 values and need parallel updates. |

Items 1 to 4 are mechanical but non-trivial; items 5 to 7 are protocol design
work shared with the host side.

## The mixed V1/V2 display question

The device renders one amount format. While V1 and V2 tokens coexist, the
device must know, per token, how many decimals to render, or it will show a
wrong human-readable amount for one of the two kinds. That information has to
travel through the token registration flow (the version byte in item 7) and be
stored per token in RAM, which is a small cost (1 byte per token) but a
mandatory protocol change to the SIGN_TOKEN_DATA / SEND_TOKEN_DATA APDUs and to
the host's cached token signatures, which are all invalidated.

# What V2 breaks on the host
[host-impact]: #host-impact

| # | Location | Problem |
| --- | --- | --- |
| 1 | wallet-lib `getDataToSign()` | The sighash bytes the device parses are produced by the wallet-lib. V2 output encoding lands there first; the Ledger path consumes it automatically, which is exactly why an un-updated app must be fenced off. |
| 2 | `hathor-wallet/src/utils/ledger.js:153` | The sign-tx protocol byte is hardcoded to `0x01`; needs the V2 value and gating logic. |
| 3 | `hathor-wallet/src/constants.js:204` | `LEDGER_MAX_VERSION = '2.0.0'` (exclusive). **A Ledger app released as 2.x is rejected by every desktop wallet currently in the field.** Any app major bump must be coordinated with a desktop wallet release that raises the ceiling first. |
| 4 | Token signature cache | Bumping the token record version invalidates stored token signatures; users must re-register custom tokens on the device. A one-time, user-visible migration. |
| 5 | `public/ledger.js` | Minimal change. It is a dumb APDU pipe; it only changes if APDU framing changes. |

The host-side version gate (item 3) cuts both ways: it is also the **mechanism
that protects users during the transition**. The desktop wallet can detect an
old app via GET_VERSION and refuse to build V2 transactions for it, keeping
Ledger users on V1 until their app is updated.

# Options
[options]: #options

Three realistic paths, presented without a recommendation:

| | Option 1: Minimal patch | Option 2: Modernize | Option 3: Fence off V2 |
| --- | --- | --- | --- |
| What | Patch the existing BAGL app: new value parser, decimals-aware formatter, protocol v2, per-token decimals, tx_version validation | Rebuild on the standard-app SDK with NBGL: same V2 work plus Stax/Flex support, dropping Nano S; opportunity to add missing features (authorities, nano contracts) | No app change. Host refuses V2 operations on Ledger wallets; Ledger users keep a V1-only experience |
| Devices | Keeps Nano S | Drops Nano S, adds Stax/Flex | Unchanged |
| Closes feature gaps | No | Optionally | No |
| Technical debt | Increases (more code on a dead-end SDK) | Resolved | Frozen |
| User impact | Token re-registration; app update required | App update required; Nano S users lose support | Ledger users excluded from V2 tokens/amounts indefinitely |
| Effort (dev-days) | ~15 to 25 | ~40 to 70 (V2 scope only; missing features extra) | ~3 to 5 (host-side gating only) |
| External dependency | Ledger release/review pipeline | Ledger release/review pipeline (full review for a new-architecture app) | None |

Notes on the estimates:

- Figures are dev-days (1 engineer-week = 4 dev-days) of focused work and
  **exclude Ledger's review and publication pipeline**, which is calendar time
  measured in weeks to months and is the dominant schedule factor for options 1
  and 2. The app's "pending security review" status suggests the pipeline has
  historically been a bottleneck for this app.
- Option 1 and 2 estimates assume the core's V2 encoding decision is final;
  embedded parser work cannot reasonably start before that.
- Option 3 is not mutually exclusive with 1 or 2: it is also the **mandatory
  interim state** while an updated app works through Ledger review, because
  fielded devices cannot be force-updated. Some host-side gating work happens
  in every scenario.
- If V2 values stay within 2^63 in practice (depends on the cap chosen by the
  core encoding decision), item 2 of the break points shrinks from "bignum
  rendering" to "wider buffers", removing the riskiest part of option 1.

# Security consideration
[security]: #security

Regardless of the option chosen, two things should happen:

1. **The host must gate by app version before sending any V2 transaction.** The
   existing GET_VERSION handshake and the desktop wallet's min/max version
   constants are the right mechanism; this is cheap and must ship with or
   before the first V2-capable wallet-lib release.
2. **Any app update should validate `tx_version` and reject unknown values**,
   so that the *next* format change fails closed on the device instead of
   relying purely on host-side discipline. Today's app fails open (item 6 of
   the break points), and only host gating stands between an old device and a
   mis-displayed amount.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Does the network migration plan keep accepting V1-encoded transactions long
  term? Option 3's viability (and the urgency of options 1/2) depends directly
  on this.
- What is the value cap in the final core encoding decision? It determines
  whether the device needs bignum rendering or just wider buffers.
- Is Nano S support a requirement or can it be dropped? This decides between
  options 1 and 2 more than any technical factor.
- Should an app update also close the existing feature gaps (authorities, nano
  contracts), folding this into a larger Ledger roadmap discussion that is out
  of scope here?
