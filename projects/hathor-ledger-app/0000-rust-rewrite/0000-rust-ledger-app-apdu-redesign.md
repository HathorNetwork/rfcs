- Feature Name: rust_ledger_app_apdu_redesign
- Start Date: 2026-06-16
- Author: <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

This RFC proposes rebuilding the Hathor Ledger hardware-wallet application from scratch in Rust (on
LedgerHQ's official `ledger_device_sdk`), with a redesigned APDU command protocol. The new protocol is
a deliberate, breaking redesign rather than a port: it replaces the current per-device-secret token
scheme with live user-confirmed token identification, makes the device **verify every transaction input
against its previous transaction** (closing a class of mint/melt attacks the current app is blind to),
and replaces "sign an arbitrary transaction" with **operation-based signing** — the wallet declares an
intent ("mint 100 ACME and keep the authority"), the device verifies the transaction matches that
intent exactly, and the user approves a plain-language sentence instead of decoding raw outputs. The
result is a smaller, safer command surface that covers today's features (HTR and custom-token transfers,
addresses, xpubs) plus token creation and mint/melt authority operations, with clean extension points
for confidential transactions, nano contracts, and on-chain blueprints. Estimated implementation effort:
**~50–70 developer-days** for the device app.

# Motivation
[motivation]: #motivation

The current Hathor Ledger app (v1.x, ~4.6k lines of C) has three problems that justify a redesign
rather than incremental patching:

1. **It is on a frozen platform.** It uses Ledger's deprecated SDK generation (BAGL UI,
   exception-based control flow). Ledger has frozen the C boilerplate as maintenance-only and directs
   all new development to Rust. The old SDK does not support the touchscreen devices (Stax, Flex) well
   and drops Nano S entirely. Staying on it means worsening support and friction in Ledger's review.

2. **It cannot express most of Hathor's transaction types.** It handles only basic P2PKH transfers of
   HTR and custom tokens. It cannot create tokens, cannot handle mint/melt authority operations, and
   has no path to confidential transactions, nano contracts, or on-chain blueprints — all of which are
   on Hathor's roadmap.

3. **It has concrete security gaps**, two of which are serious:
   - *Token-symbol spoofing.* A transaction carries a token's 32-byte ID, not its symbol. The current
     app uses a per-device secret to authenticate token metadata, which is non-portable (it dies on
     device reset), opaque, and shows the user a 64-character hex ID that is impractical to verify.
   - *Input blindness (the critical one).* A Hathor transaction input references a previous output by
     ID only — it carries no amount and no token. The device therefore **cannot see what it is
     spending or compute the transaction's balance.** Today this means the app cannot tell a transfer
     from a mint, cannot detect a melt (which can destroy tokens with no visible output), and **cannot
     refuse to spend a mint/melt authority** because it cannot even tell that an input *is* one. As
     soon as authority operations are supported, this becomes exploitable: a malicious companion app
     could trick a user into authorizing a mint or burning their tokens while the screen shows an
     innocent transfer.

The expected outcome is a hardware-wallet app that is on Ledger's supported path, covers the features
Hathor needs now, leaves room for the roadmap, and — most importantly — only ever asks the user to
approve transactions whose full meaning the device has independently verified.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The app speaks a small set of commands over USB/Bluetooth (the "APDU" protocol). There are five:

- **GetVersion / GetAppName** — identify the running app and its version.
- **GetPublicKey** — return an address or an extended public key (xpub) for a derivation path, with an
  optional on-device confirmation screen. One command now serves both the legacy address format and the
  new confidential-transaction (CT) address format.
- **RegisterToken** — teach the device that a given 32-byte token ID corresponds to a human symbol/name,
  *after the user confirms it on the trusted screen*.
- **SignTx** — verify and sign a transaction.

Three ideas define the design.

### Idea 1: Tokens are identified by live confirmation, not a stored secret

Because a token's symbol is just a label (anyone can mint a token called "USDT"), the only ground truth
is its 32-byte ID. So the device never *assumes* it knows a token. The wallet calls **RegisterToken**;
the device shows the user the claimed symbol, name, and the canonical token ID, and the user approves.
The binding is then remembered **in volatile memory for the session only** — no device secret, no stored
state, nothing to leak or reset. From then on, any transaction that uses that token displays the
approved symbol. A token the user has not confirmed cannot appear in a signed transaction.

Example: a wallet wanting to send "ACME" first calls RegisterToken; the user sees `Symbol: ACME`,
`Name: Acme Coin`, `Token ID: 7b2c…` and confirms. Later transfers show "Send 12.00 ACME" without
re-confirming, until the app is closed.

### Idea 2: The device verifies what it is spending

Before signing, the wallet must hand the device, for **every input**, the previous transaction it
spends — and a derivation path proving the input belongs to this wallet. The device recomputes the
previous transaction's ID, checks it matches, reads the real amount and token being spent, and confirms
the path derives to that output's address. Only then does it know what is going in.

This unlocks **balance awareness**: the device computes, per token, how much enters and leaves. From
that it can finally *see* mints (more of a token leaves than entered) and melts (the reverse), and it
can detect that an input is a mint/melt authority. The device is no longer blind.

To keep this affordable, the wallet sends only the relevant ("funds") portion of each previous
transaction plus a small hash of the rest; the device proves authenticity by recomputing the ID. A flag
reserves a future "full" mode for features whose data lives elsewhere (nanocontract calls and shielded outputs).

### Idea 3: The wallet declares an operation; the device enforces its shape

Rather than signing arbitrary transactions and hoping the user understands the outputs, the wallet
**declares an operation** in the first message — *send tokens*, *create token*, *mint*, *melt*,
*delegate authority*, *destroy authority*. Each operation has a strict, narrow shape. The device checks
the actual transaction against it: a "send-tokens" operation may not contain any authority input or
output and must conserve every token's balance; a "mint 100 ACME, keep authority" operation must mint
*exactly* 100, consume the right HTR deposit, and preserve exactly one mint authority — anything else is
rejected.

The declared amounts are **claims the device verifies**, never trusts. So the screen can say "Mint 100
ACME and keep the mint authority?" — a sentence the user actually understands — and the user can rely on
it, because the device proved the transaction does exactly that and nothing more. A transaction that
fits no operation is refused. (This version has no "blind signing"; a reserved slot exists for a future
gated advanced mode.)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

All commands use class byte `CLA = 0xE0`. Integers are big-endian; hashes/IDs are 32 bytes; BIP32 paths
are `len(1) || components(4 each)`; coin type is `280'`. Success status word is `0x9000`.

## Command inventory

| INS | Command | P1 | P2 |
|---|---|---|---|
| `0x03` | GetVersion | 0 | 0 |
| `0x04` | GetAppName | 0 | 0 |
| `0x05` | GetPublicKey | operation | type+display |
| `0x06` | RegisterToken | 0 | 0 |
| `0x08` | SignTx | phase | chunk/flags/index |
| `0x0A–0x0F` | reserved (CT signing, nano, blueprints, advanced) | — | — |

## GetVersion / GetAppName

- GetVersion → `MAJOR(1) | MINOR(1) | PATCH(1) | flags(1)` (`flags` reserved `0x00`).
- GetAppName → ASCII `"Hathor"`.

## GetPublicKey (0x05)

- `P1`: `0x00 = XPUB`, `0x01 = ADDRESS`. `P2`: `bit0` = address type (`0` legacy / `1` CT), `bit7` =
  confirm-on-device. Data = BIP32 path.
- Legacy XPUB → `pubkey(33) | chain_code(32) | parent_fingerprint(4)`.
- Legacy ADDRESS → `len(1) | base58 P2PKH address`.
- CT XPUB → a scan block and a spend block (each `pubkey(33)|chain_code(32)|fingerprint(4)`), derived
  from the fixed scan account `m/44'/280'/1'` and spend account `m/44'/280'/2'`.
- CT ADDRESS → `len(1) | base58 shielded address`, encoding `network_byte | scan_pubkey(33) |
  spend_pubkey(33) | checksum(4)`.

The CT scan key is a view-only key (it can detect incoming confidential payments but not spend);
exposing its public key leaks no view ability. CT *signing* is out of scope for this version.

## RegisterToken (0x06)

Data: `version(1)=0x01 | uid(32) | symbol_len(1) | symbol | name_len(1) | name`, symbol ≤ 5 and name ≤
30 printable-ASCII bytes. The device normalizes symbol/name (strip whitespace, lowercase) to reject
look-alikes; refuses anything colliding with an existing token or with the reserved `HTR`/`Hathor`;
treats a re-registration of the same token as a silent success; shows `Symbol / Name / Token ID` for
approval; and on approval caches the binding in a volatile session registry (16–32 entries, LRU, cleared
on app close). Response is empty.

## SignTx (0x08)

`P1` is the phase: `INIT(0) → VERIFY_INPUTS(1) → STREAM(2) → SIGN(3) → DONE(4)`.

- **INIT** declares the operation and its parameters, the change outputs (output index + path), a header
  descriptor (0 in this version), and the input count. Resets context.
- **VERIFY_INPUTS** carries one envelope per previous transaction: `mode | nonce | [output_index,
  owner_path]… | funds_bytes | sha256(rest)`. The device recomputes and matches the previous
  transaction ID, reads each spent output's amount/token, proves ownership via the path, and accumulates
  per-token input totals and authority flags. (`mode = FULL` is reserved → rejected for now.)
- **STREAM** carries the exact canonical signing bytes, hashed verbatim. The device parses them to
  display outputs, cross-checks every input against the verified set, rejects non-canonical encodings,
  hides verified change, and accumulates per-token output totals.
- On completion the device runs the **operation validator** and shows the operation confirmation.
- **SIGN** (`P2 = input index`) returns one signature per input, signed with that input's
  already-verified path. **DONE** wipes state.

Signature digest: `SHA-256` of the canonical signing serialization, signed with deterministic ECDSA
(RFC 6979) over secp256k1 — matching Hathor's `get_sighash_all_data`. The same digest signs all inputs.

## Operations (PROVISIONAL — pending product sign-off)

| op | declares | device verifies | screen |
|---|---|---|---|
| send-tokens | — | no authority in/out; every token balances; P2PKH outputs only | "Send X A to …" |
| create-token | name, symbol, mint amount | token-creation shape; correct HTR deposit; initial mint = declared | "Create ACME, mint 100" |
| mint-tokens | token, amount, keep-authority | mint authority spent; net mint = amount; deposit consumed; authority kept iff requested | "Mint 100 ACME; keep authority?" |
| melt-tokens | token, amount, keep-authority | melt authority spent; net melt = amount; deposit refunded | "Melt 100 ACME" |
| delegate-authority | token, authority kind, recipient | authority spent; authority output to recipient | "Send MINT authority for ACME → addr" |
| destroy-authority | token, authority kind | authority spent; no matching authority output | "Destroy MELT authority for ACME" |

A transaction matching no operation is rejected. Operation value `0xFF` is reserved for a future gated
"advanced" mode. Validators encode protocol constants (deposit ratio, authority bitmaps) pinned to the
target network at build time.

## Status words (subset)

`0x9000` OK · `0x6985` user-rejected · `0x6A86/0x6A87` bad params/length · `0x6D00/0x6E00`
unknown INS/CLA · `0xB001–0xB003` token errors · `0xB013` input-verification failure · `0xB014`
operation-shape mismatch · `0xB015` unsupported feature.

## Corner cases handled

- Large output values use Hathor's negated-int64 8-byte encoding; the parser rejects non-canonical
  encodings so the displayed amount always equals the signed amount.
- A previous transaction referenced by several inputs is streamed once (list of output indices).
- A token-creation previous transaction's spent-token ID equals that transaction's own hash.
- The on-the-wire 4-byte nonce is hashed in its 16-byte form when recomputing IDs.
- Multi-party transactions (inputs not owned by this wallet) are *not* supported in this version; every
  input must prove ownership. Atomic swaps become a future operation type.

# Effort estimation
[effort-estimation]: #effort-estimation

Estimate for the **device application only** (host/wallet integration and Ledger's external review
calendar time are separate). Figures are developer-days for one engineer comfortable with embedded Rust.

| Workstream | Est. (dev-days) |
|---|---|
| Freeze the APDU spec from this design (lock operation params, status words, limits) | 2–3 |
| App scaffold from the Rust boilerplate: dispatcher, CLA/INS routing, BIP32/crypto glue, settings, menus, multi-device build + CI | 4–5 |
| GetVersion + GetAppName | 0.5–1 |
| GetPublicKey: legacy xpub + P2PKH address + CT scan/spend xpubs + shielded address + confirm screens | 5–7 |
| RegisterToken: session registry, normalization/dedup, confirmation UI | 3–4 |
| SignTx core: streaming canonical parser (funds/inputs/outputs, value encoding, P2PKH scripts), incremental hashing, phase state machine | 6–8 |
| SignTx input verification: prevtx envelope, ID recomputation, ownership proof, per-token balance, binding rule | 5–7 |
| SignTx operation validators (6 operations) + per-operation descriptive screens | 9–12 |
| SignTx per-input ECDSA signing (by index, DER) | 1–2 |
| Cross-device NBGL polish (Stax/Flex touch review flows, snapshots) | 2–3 |
| Error model, status words, state-machine guards, hardening | 2–3 |
| Test stack: functional tests (Speculos/Ragger) per operation and per device, unit tests, fuzzing, CI | 8–11 |
| Docs, QA pass, Ledger review submission prep | 3–4 |
| **Total** | **~50–70** |

Notes for planning:
- This is **higher than a like-for-like Rust port (~30–40 dev-days)**. The delta is almost entirely the
  two security investments — always-on input verification and the six operation validators — which are
  the point of the redesign, not incidental scope.
- The operation validators and the test matrix dominate; trimming the launch operation set (e.g. ship
  send/create/mint/melt first, add the authority-delegation/destroy operations later) is the most direct
  lever to reduce initial effort.
- Roughly parallelizable across a firmware engineer and a test/QA engineer to compress wall-clock time.
- Excludes: confidential-transaction signing, nano contracts, on-chain blueprints, MultiSig, the host
  client library, and Ledger's external security review (weeks of calendar lead time — submit early).

# Drawbacks
[drawbacks]: #drawbacks

- **It is a breaking change.** Every wallet integration (and the companion libraries) must adopt the new
  protocol; there is no backward compatibility with the 1.x app.
- **Operation-based signing constrains wallets.** The wallet must build transactions that fit a
  supported operation shape; unusual but valid transactions become unsignable until an operation is
  added. This is intentional (it is what makes the screens trustworthy) but it shifts work and
  flexibility onto the wallet and couples the device to Hathor's transaction semantics, so protocol
  changes can require app updates and re-review.
- **Input verification adds latency and host complexity.** The wallet must supply every previous
  transaction, and the device hashes them — slower signing and more data to marshal than a naive signer.
- **More upfront effort** than a straight port (~50–70 vs ~30–40 dev-days).
- **Per-device, per-session token trust** means the user re-confirms a token's identity on each new
  device and after each app restart (mitigated by good wallet UX; a signed-token-list scheme is a future
  option).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

**Why this design.** Hardware-wallet security rests on the user approving what they can understand and
the device verifying what it signs. Input blindness breaks both for any feature beyond plain transfers,
so input verification is not optional once authorities are in scope — and once the device knows the
balance, operation-based signing is the natural way to turn that knowledge into screens a human can
trust. Clear-sign token registration removes a device secret and a whole class of state-management bugs
while strictly improving portability.

**Alternatives considered:**
- *Like-for-like Rust port of the 1.x protocol.* Cheaper (~30–40 dev-days) but carries forward the
  token-secret scheme and the input blindness; it cannot safely support authority operations, so it
  fails the core requirement.
- *Keep the symmetric-secret token scheme.* Rejected: non-portable, opaque, and the user still must
  verify a 64-hex ID; the live-confirmation registry is simpler and statefree.
- *On-device persistent token registry (remember across sessions).* Rejected for now: device flash is
  ample but persistence adds wear-management, eviction, and migration complexity for a marginal UX gain;
  a volatile session cache captures most of the benefit at none of the cost.
- *Generic transaction signing with raw-output display* (what most apps do). Rejected: it cannot convey
  mint/melt meaning safely and leaves the user decoding authority outputs.
- *Doing nothing.* The app stays on a frozen SDK, never gains token creation or authority support, and
  retains the input-blindness exposure.

# Prior art
[prior-art]: #prior-art

- **Input verification** mirrors Ledger's Bitcoin app, which streams previous transactions to learn
  input amounts (the pre-SegWit "trusted input" mechanism), precisely because a blind signer is
  vulnerable to fee/amount lies. Hathor has no SegWit-style amount commitment, so this approach applies
  directly.
- **Operation-based / intent-based signing** echoes "clear signing" efforts across the ecosystem
  (Ledger's transaction plugins, EIP-712 structured display) and the general lesson that users cannot
  safely approve opaque payloads.
- **Live token confirmation + signed token lists** parallels how Ledger handles ERC-20 tokens (signed
  descriptors); we adopt the live-confirmation half now and reserve the signed-list half as a future
  enhancement.
- **View-key / spend-key separation** for confidential transactions follows Monero's model (the Monero
  Ledger app exports a view key for scanning while the spend key stays on device).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **The operation set and its parameters** (the table above) need product sign-off; some operations
  likely want richer parameters (multi-recipient mint, partial authority splits, timelocks).
- A few GetPublicKey details for CT addresses (exact path-input encoding given the fixed scan/spend
  accounts; whether xpub export should always force a confirmation).
- Final numbering of application-specific status words; confirmation of token symbol/name length limits
  against hathor-core settings.
- Whether to ship the full six-operation set at launch or stage the authority operations.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Confidential transactions:** the address model and a reserved "full previous-transaction" mode are
  already designed in; signing confidential outputs (verifying Pedersen commitment openings on-device,
  which is feasible in Rust) and a gated view-key export for scanning are the natural next steps.
- **Nano contracts and on-chain blueprints:** these attach as transaction headers; the streaming parser
  and reserved INS leave room, with method calls displayed structurally and opaque arguments handled by
  a future gated advanced-signing mode.
- **Signed token lists** for well-known tokens (portable, no per-session confirmation) and a
  standardized short word-fingerprint for token IDs to make human verification easier.
- **Multi-party operations** (atomic swaps) as new operation types that relax the all-inputs-owned rule.
- **A host client library** (e.g. on Ledger's Device Management Kit) targeting this protocol, so wallet
  teams integrate against a maintained SDK rather than raw APDUs.
- **Capability negotiation** via the reserved GetVersion `flags` byte, letting wallets detect which
  operations and features a given app build supports.

