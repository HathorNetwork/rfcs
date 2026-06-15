# Effort Estimation: Token Amount v2 Serialization in the Hathor Ledger App

## TL;DR

**Estimated effort: ~7–8 developer-days (≈1.5 weeks for one developer)**, plus Ledger's
external app-review cycle (calendar weeks, low effort on our side). The wire-format change
itself is small and well-localized, but one consequence dominates the estimate: **v2 amounts
can be up to 15 bytes (117 bits), so the app's `uint64_t` value representation and all
decimal formatting must be rewritten for 128-bit arithmetic with 18 decimal places.**
Dropping v1 support entirely (as planned) keeps this from being worse — there's no
dual-path code.

## The v2 protocol (from hathor-core PR #1706 / RFC #113)

The encoding is a length-prefixed big-endian integer:

| Field   | Size           | Rules                                                             |
| ------- | -------------- | ----------------------------------------------------------------- |
| length  | 1 byte         | 1–15 (`0x0F` max); 0 only allowed in non-strict mode (zero value) |
| payload | `length` bytes | Big-endian, canonical (first byte must not be `0x00`), value > 0  |

- **Max value:** `2^63 × 10^16` = 92,233,720,368,547,758,080,000,000,000,000,000 (15 bytes / 117 bits)
- **Decimals:** 18 places instead of v1's 2 (same max *token* amount as v1, 16 extra digits of precision)
- Decoder must reject: length > 15, leading-zero payloads (non-canonical), values above max, truncated payloads, and (strict) zero values.

## Current state of the ledger app

The value handling is tightly localized — `output.value` is read in exactly one place and consumed in exactly one place:

- **Parse:** `parse_output_value()` in `src/transaction/deserialize.c:41-64` decodes the v1 4-or-8-byte sign-bit format into a `uint64_t`.
- **Display:** `format_value()` in `src/common/format.c:147-176` formats that `uint64_t` with hardcoded 2 decimals into a 30-char buffer (`display.c:331`).
- The signing flow (`sign_tx.c`) only needs the parser to consume the right number of bytes; the signature is over the raw stream, so it's unaffected beyond the decode call.
- Tests: Python integration client (`tests/app_client/transaction.py` mirrors the v1 encoding), CMocka unit tests, and a libFuzzer tx-parser harness (`fuzzing/fuzz_tx_parser.cc`) all encode v1 amounts and need updating.
- Targets: Nano S, Nano S+, Nano X (`ledger_app.toml`).

## Key technical findings (cost drivers)

1. **`uint64_t` is no longer sufficient.** `tx_output_t.value` must become a 128-bit representation (two `uint64_t`s or a 16-byte big-endian array — the Ledger SDK has no native `__int128` guarantee across targets). The decode side is easy; the cost is downstream.
2. **Decimal formatting becomes the hardest task.** Rendering a 117-bit integer as a decimal string requires big-number div/mod-by-10 (or by 10^9 in limbs) — `format_value()` and `format_fpu64()` are both 64-bit-only. Add 18 decimal places, thousands separators, and the display string grows to ~45+ chars: the `g_amount[30]` buffer must grow, and on-screen pagination behavior on Nano S needs verification. A product decision is needed on trailing-zero trimming (showing `1.000000000000000000 HTR` is hostile UX).
3. **Strictness should mirror the fullnode.** The canonical-encoding check matters on a signer: two byte-encodings of the same value would produce different tx hashes, so the device should reject non-canonical input rather than normalize it.
4. **Memory impact is minor.** +8 bytes × 10 stored outputs ≈ 80 B RAM; v2 encoding is actually *smaller* on the wire for typical amounts, so the 300-byte chunk buffer is fine. Nano S (the tightest target, and deprecated in newer Ledger SDKs) should still fit.

## Task breakdown

| # | Task                        | Scope                                                                                                                                                                                                                                                                 | Estimate                                    |
|---| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| 1 | **Spec & design**           | Confirm with the fullnode team how v2 is signaled on the wire (tx version / decimal-version field — defined in the earlier PRs in this series and RFC #113); decide zero-value and authority-output handling; decide 18-decimal display policy (trim trailing zeros?) | 0.5 d                                       |
| 2 | **Value type migration**    | Replace `uint64_t value` in `tx_output_t` with a 128-bit type; ripple through `deserialize.c`, `sign_tx.c`, `display.c`; remove v1 parse path                                                                                                                         | 1 d                                         |
| 3 | **v2 decoder**              | Rewrite `parse_output_value()`: length prefix, bounds (1–15), canonical check, max-value check, partial-chunk handling (value can now span APDU chunk boundaries at any length 2–16 bytes)                                                                            | 1 d                                         |
| 4 | **Display & formatting**    | 128-bit → decimal string conversion; 18-decimal formatting with separators; resize `g_amount`; verify pagination/rendering on all 3 device targets in Speculos                                                                                                        | 2 d                                         |
| 5 | **Tests**                   | CMocka unit tests for decoder edge cases and the new formatter; update fuzz harness + seed corpus; rewrite Python client `serialize_value()`/`from_bytes()` to v2; update all integration tests and `qa.py`; regenerate any golden screens                            | 2–3 d                                       |
| 6 | **Docs & release prep**     | `doc/TRANSACTION.md` output format, `COMMANDS.md` if version signaling changes, CHANGELOG, app version bump (this is a breaking change — major bump)                                                                                                                  | 0.5 d                                       |
| 7 | **Ledger review & release** | Submission to Ledger for security review and app-store deployment                                                                                                                                                                                                     | low effort, **weeks of calendar lead time** |

**Total: ~7–8 dev-days.** Tasks 2–3 and 4 can be parallelized across two developers, compressing wall-clock dev time to under a week.

## Risks & dependencies

- **Ledger review lead time** is out of our control.
- **\[Open question\] version signaling:** The device-side dispatch ("this tx uses v2 amounts") depends on how the version/decimal-version field is serialized, defined in the earlier parts. This must be pinned down in task 1 before task 3 starts.
- **128-bit formatting correctness** is the main defect risk, it's hand-rolled arithmetic on a signing device. Budget for exhaustive unit tests and fuzzing (already counted in task 5).
