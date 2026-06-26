- Feature Name: shielded_outputs
- Start Date: 2026-03-03
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Hathor Labs

# Summary
[summary]: #summary

This RFC introduces **shielded outputs** for Hathor transactions — a header-based extension that hides output amounts and, optionally, token types using Pedersen commitments, range proofs, and asset surjection proofs. Rather than defining a new transaction version, shielded outputs attach to existing transaction types (regular transactions and token creation transactions) via a set of transaction headers, preserving full backward compatibility. Two privacy tiers are offered: `AmountShieldedOutput` (amount hidden, token visible) and `FullShieldedOutput` (both amount and token hidden). Recipients recover hidden values through ECDH-based range proof rewinding, requiring no out-of-band communication.

The design covers four headers: `ShieldedOutputsHeader` (carries the shielded outputs), `UnshieldBalanceHeader` (carries the balance residual for full-unshield transactions), and `MintHeader` / `MeltHeader` (publicly declare per-token supply changes so that mint and melt operations may participate in shielded transactions while remaining auditable). This document is self-contained: it specifies the full data model, cryptography, transaction rules, verification pipeline, fee mechanism, and supply-audit properties as actually implemented.

> **Implementation status.** This RFC documents the implementation on branch
> `feat/shielded-outputs-rebased` of `hathor-core`. Where the design differs from
> earlier drafts (range proof construction, balance-residual representation, fee
> rates, library layout), this document reflects the **code**, and the earlier
> framing is preserved only under [Future possibilities](#future-possibilities)
> when it describes work not yet implemented.

# Motivation
[motivation]: #motivation

**Privacy is a fundamental property of money.** Physical cash does not broadcast the denomination of every bill exchanged between two parties, yet transparent blockchains do exactly that. Every Hathor transaction today reveals the exact amount transferred, the token involved, and — combined with address analysis — a detailed picture of economic activity.

This transparency creates real problems:

- **Payroll and compensation.** An employer paying salaries on-chain reveals every employee's compensation to anyone who looks.
- **Business-to-business payments.** Suppliers, vendors, and partners can reverse-engineer pricing, margins, and deal terms from on-chain flows.
- **Trading and DeFi.** Front-runners and MEV extractors exploit visible amounts to sandwich trades or copy strategies.
- **Personal finance.** Any recipient of a payment can trace the sender's full balance and transaction history.
- **Multi-token privacy.** Hathor supports custom tokens. When token types are visible, observers can track the flow of specific assets (e.g., loyalty points, governance tokens, stablecoins), revealing business relationships and portfolio composition.
- **Confidential issuance.** A token issuer who routinely transacts in shielded form should be able to mint or melt without dropping to fully-transparent mode, while still letting anyone audit total supply.

Shielded outputs address these problems by making amounts and token types cryptographically opaque to everyone except the transaction participants, while preserving the ability of every full node to verify that no inflation or double-spending has occurred — and the ability of anyone to audit per-token supply.

**Expected outcome:** Users can opt into amount privacy (and optionally token-type privacy) on a per-output basis, within ordinary Hathor transactions, with no protocol-level changes to transaction versions, input formats, or the DAG structure. Issuers can mint, melt, and create tokens directly into shielded outputs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## 1. What Are Shielded Outputs?

A shielded output replaces the plaintext `(amount, token)` pair in a standard `TxOutput` with a cryptographic **commitment** — a value that provably encodes the correct amount and token without revealing either.

```
Standard output:       100 HTR to address H7bKm...
                       ^^^ visible to everyone

Amount-shielded:       [commitment] HTR to address H7bKm...
                       ^^^^^^^^^^^          ^^^ token still visible
                       amount hidden

Fully shielded:        [commitment] [asset commitment] to address H7bKm...
                       ^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^
                       amount hidden  token hidden
```

Hathor offers **two privacy tiers**, selectable per output:

| Tier | Hides Amount | Hides Token | Proof Overhead | Use Case |
|------|:---:|:---:|---|---|
| `AmountShieldedOutput` | Yes | No | Range proof (~3213 B) | Hide salary amounts while token type is public |
| `FullShieldedOutput` | Yes | Yes | Range proof + surjection proof (~3213 + variable B) | Full privacy for multi-token transactions |

Both tiers use **Pedersen commitments** for amounts and **Borromean range proofs** to guarantee amounts are non-negative (specifically in `[1, 2^40)`). `FullShieldedOutput` additionally uses **blinded asset tags** and **asset surjection proofs** to hide and validate the token type.

### The Privacy Stack

Shielded outputs are Phase B of Hathor's three-phase privacy roadmap:

```
+-----------------------------------------------------------+
|  Phase C: INPUT UNLINKABILITY (future)                    |
|  Ring signatures or nullifiers                            |
|  "Which output was spent?" -> Hidden                      |
+-----------------------------------------------------------+
+-----------------------------------------------------------+
|  Phase B: SHIELDED OUTPUTS  <- THIS RFC                   |
|  Pedersen commitments + range proofs + surjection proofs  |
|  "How much?" + "Which token?" -> Hidden                   |
+-----------------------------------------------------------+
+-----------------------------------------------------------+
|  Phase A: ADDRESS PRIVACY (Silent Payments RFC)           |
|  ECDH-derived one-time addresses                          |
|  "Who received it?" -> Hidden                             |
+-----------------------------------------------------------+
```

Each phase is independently useful and composes with the others. With all three active, an observer learns almost nothing about a transaction beyond the number of inputs and outputs.

## 2. How It Works

### Shielding: Moving Funds into Privacy

To shield funds, a user creates a transaction with transparent inputs and shielded outputs:

```
BEFORE (transparent):
  Alice has 100 HTR (visible on-chain)

SHIELDING TRANSACTION:
  Input:   100 HTR (transparent, amount visible)
  Output0: [commitment_A] (shielded, 90 HTR hidden inside)
  Output1: [commitment_B] (shielded, 5 HTR hidden inside)
  Fee:     5 HTR (always transparent)

AFTER:
  Alice has two shielded UTXOs totaling 95 HTR
  Observer sees: "100 HTR entered the shielded pool"
  Observer does NOT know: the split between Output0 and Output1
```

Note: at least 2 shielded outputs are required whenever a transaction has any shielded outputs (Rule 4 — see Section 4.3). This prevents an observer from trivially deducing a single output's amount.

### Unshielding: Returning to Transparency

To unshield, a user spends shielded inputs into transparent outputs. There are two shapes:

**Partial unshield** — at least one shielded output remains (e.g. shielded change). The balance equation is the ordinary one (`sum(C_in) == sum(C_out) + fee·H_HTR`), and the wallet makes the last shielded output absorb the blinding residual.

**Full unshield** — *no* shielded outputs remain (every output is transparent). There is no shielded output to absorb the blinding residual, so the transaction carries an **`UnshieldBalanceHeader`** holding the scalar `excess = sum(r_in) − sum(r_out)`, and the balance equation becomes `sum(C_in) == sum(C_out) + excess·G + fee·H_HTR`.

```
FULL-UNSHIELD TRANSACTION:
  Input:   [commitment] (shielded, 90 HTR hidden)
  Output0: 85 HTR to Bob (transparent)
  Fee:     5 HTR (transparent)

  Header: UnshieldBalanceHeader { excess_blinding_factor (32B scalar) }

Observer sees: 85 HTR left the shielded pool
Observer does NOT know: the shielded input amount (other than that it equals 85 + 5)
```

> **Privacy footgun.** The scalar form of `UnshieldBalanceHeader` exposes
> `sum(r_in)` of the unshielded inputs as a raw scalar. A point-form replacement
> (`E_tx = excess·G` plus a Schnorr binding signature) closes this; it is **not
> implemented** and is described in [Future possibilities](#future-possibilities).

### Mixed Transactions

Transparent and shielded inputs/outputs can be freely combined in a single transaction. The balance equation works uniformly: transparent amounts are treated as "trivial commitments" with a zero blinding factor, and the homomorphic verification covers everything in one check.

| Type | Transparent In | Shielded In | Transparent Out | Shielded Out | Balance header | Use Case |
|------|:---:|:---:|:---:|:---:|---|---|
| Standard | Yes | — | Yes | — | — | Legacy transaction |
| Shielding | Yes | — | Optional | Yes (≥2) | — | Enter shielded pool |
| Confidential | — | Yes | — | Yes (≥2) | — | Fully private transfer |
| Partial unshield | — | Yes | Yes | Yes (≥2) | — | Exit, keep shielded change |
| Full unshield | — | Yes | Yes | — | `UnshieldBalanceHeader` | Exit shielded pool entirely |
| Fully mixed | Yes | Yes | Yes | Yes (≥2) | — | Maximum flexibility |

A shielded transaction must carry **exactly one** of `ShieldedOutputsHeader` or `UnshieldBalanceHeader` — never both and never neither (enforced inside balance verification).

### Confidential Mint, Melt, and Token Creation

A shielded transaction may exercise mint or melt authority and may create new tokens directly into shielded outputs. The per-token supply change is declared publicly via two headers:

- **`MintHeader`** — list of `(token_index, amount)` entries declaring supply *created* in this transaction.
- **`MeltHeader`** — list of `(token_index, amount)` entries declaring supply *destroyed* in this transaction.

The declared amounts are bound into the homomorphic balance equation as public scalar terms (Rule M4), so the verifier enforces supply correctness without learning where the new value lands or which UTXO was melted. Because the amounts are public, total per-token supply remains computable by anyone. What becomes private is the *recipient set*, the *per-recipient amounts*, and *which shielded UTXO is melted*.

A given token may appear in **either** `MintHeader` **or** `MeltHeader` in a single transaction, never both (Rule M3). Worked examples appear in [§2.7](#27-mintmelt-examples).

### Fee Model

Shielded outputs impose additional verification costs (range-proof verification, surjection-proof verification, larger storage). A per-output fee compensates the network:

- `FEE_PER_AMOUNT_SHIELDED_OUTPUT`: charged per `AmountShieldedOutput` (default: `1` HTR base unit)
- `FEE_PER_FULL_SHIELDED_OUTPUT`: charged per `FullShieldedOutput` (default: `2` HTR base units)

Fees are declared in the existing `FeeHeader`, are fully transparent, and are burned. See [Section 4.6](#46-fee-mechanism) for details.

### What Remains Visible

Even with shielded outputs, the following information is always public:

- **Transaction structure**: number of inputs, number of outputs, which outputs are shielded vs. transparent.
- **Fee amounts**: always plaintext HTR (Rule 2).
- **Authority outputs**: mint/melt authority tokens are always transparent (Rule 7).
- **Per-token supply delta**: each `(token_index, amount)` declared in `MintHeader`/`MeltHeader`.
- **HTR deposit/withdraw**: for `DEPOSIT`-version tokens, derived from the declared mint/melt amounts via Hathor's 1% rule.
- **Scripts**: the locking script (recipient address) remains visible unless combined with Silent Payments (Phase A).
- **Transparent inputs/outputs**: their amounts and tokens remain fully visible.

## 3. Shielded Addresses (Pending Decision)

Receiving shielded outputs requires the sender to know the recipient's full public key (not just the address hash) in order to establish the ECDH shared secret for blinding factor communication. Today, P2PKH addresses only encode a hash of the public key.

A new **shielded address type** would bundle the necessary keys into a single address the recipient can publish.

### Key Components

A shielded address encodes two keys:

- **Scan public key** (`B_scan`, 33 bytes): Used for ECDH — the sender computes a shared secret with this key, and the recipient uses the corresponding private key to recover blinding factors.
- **Spend public key** (`B_spend`, 33 bytes): Controls spending authority. The output script locks funds to `hash(B_spend)` (standard P2PKH).

> **On-chain representation:** The output script remains a standard P2PKH locking to `hash(B_spend)`. The scan public key is never stored on-chain — it is only known to the sender (from the shielded address) and the recipient.

### Candidate Formats

Three formats are under consideration:

#### Option A: Compact (53 bytes of data)

```
scan_pubkey(33) || hash(spend_pubkey)(20)
```

The spend pubkey is represented as a 20-byte hash (like a standard P2PKH address). Shorter addresses, and fully sufficient for shielded outputs (ECDH uses the scan key; output locking uses the spend key hash). However, the full spend pubkey is not recoverable from the address alone, which prevents future Silent Payments integration (one-time address derivation requires EC point addition on `B_spend`).

#### Option B: Full Keys (66 bytes of data)

```
scan_pubkey(33) || spend_pubkey(33)
```

Both keys are present in full. Longer addresses, but the sender has everything needed without any additional lookups. Compatible with BIP352-style Silent Payments, where the spend pubkey is required for one-time address derivation.

#### Option C: Spend Key Only (33 bytes of data)

```
spend_pubkey(33)
```

Only the spend public key is encoded — no scan key at all. This is the shortest possible shielded address and the simplest to implement. However, without a scan key, the wallet cannot delegate blockchain scanning to a third-party service (the scan private key is what enables a service to detect incoming payments without spending authority). The wallet must download and trial-decrypt the entire shielded transaction history itself to compute its balance, which is impractical for light clients.

### Tradeoff Analysis

| Aspect | Option A (Compact) | Option B (Full Keys) | Option C (Spend Only) |
|--------|---|---|---|
| Address length | ~90 characters | ~115 characters | ~55 characters |
| Data payload | 53 bytes | 66 bytes | 33 bytes |
| Spend pubkey recovery | Requires lookup or out-of-band | Self-contained | Self-contained |
| Silent Payments compat. | No (need full spend pubkey for ECDH) | Yes | No (no scan key for ECDH) |
| QR code density | Lower (smaller) | Higher (larger) | Lowest (smallest) |
| Delegated scanning | Yes (scan key separates detection from spending) | Yes (scan key separates detection from spending) | No (wallet must scan entire history itself) |
| Light client support | Yes | Yes | No |

**Status:** Decision pending. The implementation currently uses the full public key for ECDH, so Option B aligns with the existing code path.

## 4. Wallet Integration Guide

### Receiving Shielded Payments

Every shielded output contains an `ephemeral_pubkey` field (33 bytes; all-zero means "not present"). The recipient recovers the hidden amount through **ECDH + range proof rewind**:

1. **Address match**: The wallet checks if the output script contains `hash(spend_pubkey)` matching one of its keys.
2. **ECDH shared secret**: `s = ECDH(scan_privkey, ephemeral_pubkey)` (secp256k1 ECDH; SHA256 of the shared point).
3. **Nonce derivation**: `nonce = SHA256("Hathor_CT_nonce_v1" || s)`.
4. **Range proof rewind**: Using the nonce as the rewind key, the wallet calls `rewind_range_proof()` which returns the committed `(value, blinding_factor, message)`.
5. **For `FullShieldedOutput`**: The `message` field contains `token_uid(32B) || asset_blinding_factor(32B)`. The wallet reconstructs the expected asset commitment from these values and cross-checks against the on-chain `asset_commitment` to prevent a malicious sender from claiming a worthless token is HTR.

If rewind fails (wrong nonce), the output is not addressed to this wallet — skip it silently.

### Sending Shielded Payments

1. **Choose privacy tier** per output (`AmountShieldedOutput` or `FullShieldedOutput`).
2. **Generate ephemeral keypair** `(e, E = e*G)` for each output.
3. **Compute ECDH shared secret** with the recipient's scan public key (`B_scan`).
4. **Derive nonce** for range proof construction.
5. **Create Pedersen commitment**: `C = amount * H_token + blinding * G` (with `H_token` the blinded asset generator for `FullShieldedOutput`).
6. **Create range proof** using the deterministic nonce (enables recipient rewind).
7. **For `FullShieldedOutput`**: Create the blinded asset commitment and surjection proof, and embed `token_uid || asset_blinding_factor` in the range proof message.
8. **Balance blinding factors**: Assign random blinding factors to all shielded outputs except the last, which receives the balancing residual computed by `compute_balancing_blinding_factor()`. (For full-unshield transactions there is no last shielded output, so the residual is published as the `UnshieldBalanceHeader` scalar.)
9. **Compute and attach fee** in `FeeHeader`; attach `MintHeader`/`MeltHeader` if minting or melting.

### Blinding Factor Management and Backup

The wallet must store blinding factors for every owned shielded UTXO — without them, the funds cannot be spent.

**Recovery guarantee**: Since blinding factors for received outputs are derived deterministically from `ECDH(scan_privkey, ephemeral_pubkey)` — and both the scan private key (derivable from the seed) and the ephemeral pubkey (stored on-chain) are always available — a **seed backup is sufficient** for full recovery. The wallet re-derives all blinding factors by scanning the blockchain and rewinding the range proofs.

For change outputs, wallets should use deterministic derivation from the input blinding factors and output index to ensure recoverability.

### Balance Display

From the wallet owner's perspective, balances are fully accurate:

- **Total balance**: sum of transparent UTXOs + sum of decrypted shielded UTXOs.
- **Per-token breakdown**: the wallet knows which token each shielded UTXO holds.
- **Shielded vs. transparent breakdown**: optionally shown per token.

Privacy only affects external observers — the wallet owner always sees exact amounts.

### Rule 4 Compliance

Whenever a transaction has any shielded outputs, the wallet must include at least 2 of them. This is typically satisfied naturally (payment + change). Note that zero-value shielded outputs are forbidden by the range proof's `min_value = 1`, so in edge cases the wallet should split a real output or add a small decoy output it owns rather than emit an empty placeholder.

### Fee Computation

```
shielded_fee = (count_amount_shielded * FEE_PER_AMOUNT_SHIELDED_OUTPUT)
             + (count_full_shielded   * FEE_PER_FULL_SHIELDED_OUTPUT)
total_fee    = shielded_fee + standard_fee_if_any
```

For `DEPOSIT`-version tokens being minted/melted, the wallet must also fund (or will receive) the 1% HTR deposit/withdraw, which is folded into the balance equation rather than the `FeeHeader`. For `FEE`-version tokens, each `MintHeader`/`MeltHeader` entry costs one extra `FEE_PER_OUTPUT` (also folded into the balance equation). The wallet should display the expected fee before the user confirms.

## 5. Explorer and Indexer Impact

### What Breaks

| Capability | Status | Reason |
|---|---|---|
| Show output amounts | Broken for shielded | Hidden behind Pedersen commitments |
| Identify output token type | Broken for `FullShieldedOutput` | Hidden behind blinded asset commitments |
| Compute address balances | Broken for shielded | Cannot sum opaque curve points |
| Token supply tracking | **Unaffected** | Mint/melt amounts are declared publicly in `MintHeader`/`MeltHeader` |
| Rich list / rankings | Broken for shielded | Balances not computable |

### What Still Works

- Transaction structure (input/output count, shielded vs. transparent).
- Transparent outputs (amounts, tokens, scripts — unchanged).
- Fee amounts (always transparent).
- Cryptographic proof verification (anyone can verify correctness).
- Authority UTXO tracking (always transparent, Rule 7).
- Token metadata (name, symbol from creation transaction).
- **Per-token supply**: an indexer reconciles supply by reading `MintHeader`/`MeltHeader` entries on every shielded tx in addition to existing transparent mint/melt detection. Shielded UTXOs are skipped in per-UTXO token indexing (their amount is hidden), but supply totals come from the headers.
- **Network-wide supply audit**: `Σ C_utxo == Σ_token (total_supply · H_token)` holds for the whole chain with no protocol change. See [§4.7](#47-network-wide-supply-audit).

### Mitigations

- **Shielded pool boundary tracking**: Explorers can track aggregate amounts entering/leaving the shielded pool from the transparent side of shielding/unshielding transactions and from the mint/melt headers.
- **View key delegation**: Users can optionally share scan keys with trusted explorers for selective disclosure.
- **"Shielded" indicator**: Explorers should display shielded outputs with a clear indicator, showing commitment hex values but never placeholder amounts.

## 6. Confidential Issuance Is Auditable

Mint, melt, and token-creation transactions can now be shielded, but the supply change is always declared in plaintext via `MintHeader`/`MeltHeader`. This means:

- An explorer can see **when** mint/melt authority is exercised and **exactly how many** tokens were minted or melted.
- Token supply remains fully auditable for all custom tokens — no trust in the token creator is required.
- What is hidden is only the *recipient set*, the *per-recipient split*, and *which shielded UTXO was melted*.

## 7. Mint/Melt Examples
[27-mintmelt-examples]: #27-mintmelt-examples

### 7.1 Confidential mint

An issuer holds a transparent mint authority for token T and wants to mint 100,000 T directly into a single shielded output for a salary recipient (plus a shielded change output to satisfy Rule 4).

```
Inputs:
  - mint authority of T (transparent)
  - HTR (shielded, for deposit + fee)

Outputs:
  - mint authority of T (transparent, retained)
  - shielded HTR change (shielded)
  - shielded T output: 100,000 T (shielded)

Headers:
  - FeeHeader: standard fee
  - ShieldedOutputsHeader: 2 shielded outputs
  - MintHeader: [(token_index=1, amount=100000)]
```

Observers learn: *100,000 T were minted*. They do not learn the recipient or the issuer's HTR change.

### 7.2 Confidential token creation

An issuer creates token T with initial supply 1,000,000, distributed across 4 shielded outputs.

```
Inputs:
  - HTR (transparent or shielded), enough for 1% deposit + fee

Outputs:
  - 4 shielded T outputs (shielded)
  - transparent HTR change (transparent)

Headers:
  - FeeHeader
  - ShieldedOutputsHeader: 4 shielded outputs
  - MintHeader: [(token_index=1, amount=1000000)]
```

The token UID is the tx hash; the supply (`1,000,000`) is publicly declared. Per-recipient amounts are private. A shielded `TokenCreationTransaction` MUST declare its initial supply via exactly one `MintHeader` entry for `token_index = 1`, and MUST NOT melt the new token. (`MeltHeader` is not permitted on a token creation transaction.)

### 7.3 DEPOSIT-version token: mint with HTR deposit

Token `TD` is `DEPOSIT`-version. The issuer mints 100,000 TD across two shielded outputs. The 1% deposit rule applies: minting 100,000 TD costs 1,000 HTR. Assume shielded outputs cost 1 HTR each (`FEE_PER_AMOUNT_SHIELDED_OUTPUT`).

```
Inputs:
  - mint authority of TD (transparent)
  - HTR (transparent), 1,002 HTR available

Outputs:
  - mint authority of TD (transparent, retained)
  - 2 shielded TD outputs (total 100,000 TD)

Headers:
  - FeeHeader: 2 HTR fee (2 × FEE_PER_AMOUNT_SHIELDED_OUTPUT)
  - ShieldedOutputsHeader: 2 shielded outputs
  - MintHeader: [(token_index=1, amount=100000)]

Verifier (Rule M4):
  - deposit = 0.01 × 100,000 = 1,000 HTR, folded onto the HTR output side.
  - Augmented balance:
      sum(C_in) + 100,000·H_TD
        == sum(C_out) + 1,000·H_HTR (deposit) + 2·H_HTR (fee)
  - HTR side: 1,002·H_HTR (input) == 1,000·H_HTR + 2·H_HTR.
  - TD side: 100,000·H_TD == sum(shielded TD commitments).
```

### 7.4 FEE-version token: mint paying per-output fees

Token `TF` is `FEE`-version. The issuer mints 50,000 TF into one transparent output of 30,000 TF plus one shielded output of 20,000 TF. `FEE`-version tokens have no HTR deposit; per-output fees apply, plus one extra `FEE_PER_OUTPUT` per `MintHeader`/`MeltHeader` entry on a `FEE`-version token (Rule M4).

```
Inputs:
  - mint authority of TF (transparent)
  - HTR (shielded), enough for fees

Outputs:
  - mint authority of TF (transparent, retained)
  - transparent TF output: 30,000 TF (chargeable)
  - shielded TF output: 20,000 TF
  - shielded HTR change

Headers:
  - FeeHeader:
      1 × FEE_PER_OUTPUT (one chargeable transparent TF output)
      + 2 × FEE_PER_AMOUNT_SHIELDED_OUTPUT (two shielded outputs)
  - ShieldedOutputsHeader: 2 shielded outputs
  - MintHeader: [(token_index=1, amount=50000)]

Verifier (Rule M4):
  - No 1% deposit (FEE-version skips it).
  - Each MintHeader/MeltHeader entry on a FEE-version token contributes one
    FEE_PER_OUTPUT charge to the output side of the balance equation (folded
    directly, NOT declared in FeeHeader).
  - TF side: 50,000·H_TF balances 30,000 (transparent) + 20,000 (shielded).
  - FeeHeader total must match exactly:
      FEE_PER_OUTPUT × 1 (transparent TF output)
      + FEE_PER_AMOUNT_SHIELDED_OUTPUT × 2 (two shielded outputs).
```

The per-entry `FEE_PER_OUTPUT` is visible but leaks no per-recipient information — every entry pays exactly one `FEE_PER_OUTPUT` regardless of how many shielded recipients the entry is split across.

### 7.5 DEPOSIT-version token: melt with HTR withdraw

Token `TD` is `DEPOSIT`-version. The treasurer melts 80,000 TD; this releases 800 HTR back from the deposit pool (1% withdraw rule).

```
Inputs:
  - melt authority of TD (transparent)
  - shielded TD input (holds ≥ 80,000 TD plus optional change)

Outputs:
  - melt authority of TD (transparent, retained)
  - shielded HTR output (carries the 800 HTR withdraw + change)
  - optional shielded TD change

Headers:
  - FeeHeader: shielded fees only
  - ShieldedOutputsHeader
  - MeltHeader: [(token_index=1, amount=80000)]

Verifier (Rule M4):
  - withdraw = 0.01 × 80,000 = 800 HTR, folded onto the HTR input side.
  - Augmented balance:
      sum(C_in) + 800·H_HTR
        == sum(C_out) + 80,000·H_TD + fee·H_HTR
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 4.0 Cryptographic Library Layout

The cryptography is implemented in Rust and exposed to Python:

```
htr-rs/crates/htr-ct-crypto   Rust core (secp256k1-zkp v0.11):
                              pedersen, generators, rangeproof,
                              surjection, ecdh, balance, error
        |
        v
htr-rs/crates/htr-lib         PyO3 native extension. Registers the
                              `htr_lib.shielded` submodule.
        |
        v  (import: `from htr_lib import shielded`)
hathorlib/hathorlib/crypto/shielded/   Thin Python wrappers:
                              asset_tag.py, commitment.py, range_proof.py,
                              surjection.py, ecdh.py, balance.py, recover.py
```

`hathor-core` consumes only the `hathorlib` wrappers; it never calls `secp256k1-zkp` directly. The Python data model (output types, headers) lives in `hathorlib` and is reused by `hathor-core`.

## 4.1 Cryptographic Primitives

### Pedersen Commitments

A Pedersen commitment to a value `v` with blinding factor `r` using generator `H` is:

```
C = v * H + r * G

Where:
  v  = amount (u64)
  r  = blinding factor (32-byte scalar)
  H  = generator point specific to the token type
  G  = secp256k1 base generator
```

**Properties:** hiding, binding, and homomorphic (`C1 + C2 = (v1+v2)*H + (r1+r2)*G`). Commitments are 33 bytes (compressed point). A *trivial* commitment `C = v*H` (zero blinding) represents transparent amounts in the balance equation.

Implemented in `htr-ct-crypto/src/pedersen.rs` (`create_commitment`, `create_trivial_commitment`, `verify_commitments_sum`).

### NUMS Asset Tag Derivation

Each token has a deterministic generator `H_token` derived via a Nothing-Up-My-Sleeve (NUMS) construction (`htr-ct-crypto/src/generators.rs`):

```
tag = SHA256("Hathor_AssetTag_v1" || token_uid_32B)
H_token = Generator::new_unblinded(tag)        # 33-byte point
```

The domain separator `"Hathor_AssetTag_v1"` prevents cross-protocol collisions. The construction guarantees no one knows `x` such that `H_token = x*G`, which is essential for binding. The HTR generator is computed once and cached.

**Token UID normalization** (`hathorlib/crypto/shielded/asset_tag.py::normalize_token_uid`): Hathor uses `b'\x00'` (1 byte) for HTR and 32-byte hashes for custom tokens; the crypto library always expects 32 bytes. A 1-byte UID is right-padded with zeros to 32 bytes (`token_uid.ljust(32, b'\x00')`); a 32-byte UID passes through unchanged; any other length is an error.

### Blinded Asset Commitments

For `FullShieldedOutput`, the asset tag is blinded to hide the token type:

```
A = H_token + r_asset * G

Where:
  H_token  = unblinded NUMS generator for this token
  r_asset  = random asset blinding factor (32-byte scalar)
  A        = blinded asset commitment (33 bytes, compressed point)
```

An observer sees `A` — a random-looking curve point indistinguishable from any other token's blinded commitment.

### Borromean Range Proofs

Each shielded output includes a range proof (secp256k1-zkp's `RangeProof`, a Borromean-style proof — **not** a Bulletproof) demonstrating:

```
The committed amount v satisfies: 1 <= v < 2^40
```

(`htr-ct-crypto/src/rangeproof.rs`.)

- **Lower bound `min_value = 1`** prevents zero-amount outputs (enforced both at proof creation and re-checked at verification).
- **Fixed bit width `RANGE_PROOF_BITS = 40`.** 40 bits covers up to `2^40 − 1 ≈ 1.1 × 10^12` base units (~10 trillion HTR cents), above the maximum supply. Pinning the bit width makes every proof the **same size** regardless of the committed value, eliminating a proof-size side-channel.
- **Proof size**: ~3213 bytes; bounded by `MAX_RANGE_PROOF_SIZE = 3328`.

**Architecture: separate proofs, not aggregated.** Each output carries its own independent range proof. This supports multi-party transactions and atomic swaps (each party proves its own outputs without sharing blinding factors) and independent UTXO pruning.

> A `batch_verify_range_proofs` helper exists but is currently a sequential loop
> (`TODO` in code: investigate a true batched secp256k1-zkp API). Treat batch
> verification as a future optimization, not a current property.

### Asset Surjection Proofs

For `FullShieldedOutput` only (`htr-ct-crypto/src/surjection.rs`). Proves the output's blinded asset commitment corresponds to one of the input domain generators, without revealing which one:

```
Given:
  Domain generators:   A_1, A_2, ..., A_n   (one per input + per MintHeader entry)
  Output commitment:   A_out

A ring proof over {A_out - A_i} proves knowledge of the discrete log for exactly
one difference (the matching token), without revealing which.
```

The domain is built from: every transparent input's NUMS asset tag, every shielded input's on-chain `asset_commitment` (or derived tag for an `AmountShieldedOutput` input), and one NUMS asset tag per `MintHeader` entry (so a freshly-minted token can be the asset of a `FullShieldedOutput`). **Proof size**: grows with the domain size. Maximum: `MAX_SURJECTION_PROOF_SIZE = 4096` bytes.

### Homomorphic Balance Verification

The balance equation covers all inputs and outputs uniformly. For transactions with at least one shielded output:

```
sum(C_inputs) == sum(C_outputs) + sum(C_fee_entries)
```

For full-unshield transactions (shielded inputs, no shielded outputs) an excess scalar is added:

```
sum(C_inputs) == sum(C_outputs) + sum(C_fee_entries) + excess * G
```

Where:
- Shielded inputs/outputs use their on-chain commitment directly.
- Transparent inputs/outputs (and `FeeHeader` entries) use trivial commitments `C = amount * H_token`.
- `excess = sum(r_in) − sum(r_out)`, carried as a 32-byte scalar in `UnshieldBalanceHeader`.

Expanding via NUMS generator independence, both scalar coefficients must vanish: values balance per token, and blinding factors balance. For transactions with shielded outputs, the wallet enforces the blinding balance by construction (the last shielded output absorbs the residual via `compute_balancing_blinding_factor`). The Rust entry point is `htr-ct-crypto/src/balance.rs::verify_balance`, taking `BalanceEntry::{Transparent, Shielded}` lists plus an optional `excess`.

The mint/melt augmentation (Rule M4) is detailed in [§4.5](#45-mintmelt-reference).

## 4.2 Data Structures

### OutputMode

A 1-byte discriminator (`hathorlib/transaction/shielded_tx_output.py`):

| Value | Name | Meaning |
|---|---|---|
| `0` | `TRANSPARENT` | Standard `TxOutput` (not stored in shielded headers) |
| `1` | `AMOUNT_ONLY` | Amount hidden, token visible (no surjection proof) |
| `2` | `FULLY_SHIELDED` | Amount and token hidden (surjection proof required) |

### AmountShieldedOutput

Hides the amount; token type remains visible.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `commitment` | bytes | 33 B | Pedersen commitment `C = v*H_token + r*G` |
| `range_proof` | bytes | ~3213 B (max 3328) | Borromean range proof |
| `script` | bytes | variable (max 1024) | Locking script (P2PKH, etc.) |
| `token_data` | int | 1 B | Token index (same semantics as `TxOutput.token_data`) |
| `ephemeral_pubkey` | bytes \| None | 33 B | Compressed secp256k1 point for ECDH recovery (all-zero on the wire = not present) |

### FullShieldedOutput

Hides both amount and token type.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `commitment` | bytes | 33 B | Pedersen commitment (uses the blinded asset generator) |
| `range_proof` | bytes | ~3213 B (max 3328) | Borromean range proof |
| `script` | bytes | variable (max 1024) | Locking script |
| `asset_commitment` | bytes | 33 B | Blinded asset tag `A = H_token + r_asset*G` |
| `surjection_proof` | bytes | variable (max 4096) | Asset surjection proof |
| `ephemeral_pubkey` | bytes \| None | 33 B | Compressed secp256k1 point for ECDH recovery (all-zero = not present) |

**Typical total size**: ~3.3 KB per shielded output, dominated by the range proof.

### Size Constants

| Constant | Value | Location |
|---|---|---|
| `COMMITMENT_SIZE` | 33 | `shielded_tx_output.py` |
| `ASSET_COMMITMENT_SIZE` | 33 | `shielded_tx_output.py` |
| `EPHEMERAL_PUBKEY_SIZE` | 33 | `shielded_tx_output.py` |
| `MAX_RANGE_PROOF_SIZE` | 3328 | `shielded_tx_output.py` |
| `MAX_SURJECTION_PROOF_SIZE` | 4096 | `shielded_tx_output.py` |
| `MAX_SHIELDED_OUTPUTS` | 32 | `shielded_tx_output.py` |
| `MAX_SHIELDED_OUTPUT_SCRIPT_SIZE` | 1024 | `shielded_tx_output.py` |
| `EXCESS_BLINDING_FACTOR_SIZE` | 32 | `unshield_balance_header.py` |
| `MAX_MINT_MELT_ENTRIES` | 16 | `mint_melt_header.py` |
| `AMOUNT_SIZE` (mint/melt) | 8 (u64 BE) | `mint_melt_header.py` |

### Vertex Header IDs

All headers are ordered ascending by ID within a transaction (`hathor/transaction/headers/types.py`):

| ID | Header |
|---|---|
| `0x10` | `NANO_HEADER` |
| `0x11` | `FEE_HEADER` |
| `0x12` | `SHIELDED_OUTPUTS_HEADER` |
| `0x13` | `UNSHIELD_BALANCE_HEADER` |
| `0x14` | `MINT_HEADER` |
| `0x15` | `MELT_HEADER` |

`get_maximum_number_of_headers()` returns **5** when `ENABLE_SHIELDED_TRANSACTIONS` is active (otherwise 3), allowing `FeeHeader + (ShieldedOutputs | UnshieldBalance) + MintHeader + MeltHeader` with one slot of margin for `NanoHeader`.

### ShieldedOutputsHeader

Carries the shielded outputs; not part of the standard `tx.outputs` list.

| Field | Size | Description |
|-------|------|-------------|
| Header ID | 1 B | `0x12` |
| `num_outputs` | 1 B | Number of shielded outputs, `1 ≤ n ≤ 32` |
| Outputs | variable | Concatenated serialized outputs |

There is **no** binding signature or excess-point trailer on this header.

### UnshieldBalanceHeader

Carries the balance residual for full-unshield transactions.

| Field | Size | Description |
|-------|------|-------------|
| Header ID | 1 B | `0x13` |
| `excess_blinding_factor` | 32 B | Scalar `excess = sum(r_in) − sum(r_out)` |

Total: 33 bytes. It is a **scalar**, not a point and not a Schnorr signature. The whole serialization is bound into the sighash, so mutating the scalar invalidates all signatures. Mutually exclusive with `ShieldedOutputsHeader`.

### MintHeader / MeltHeader

Publicly declare per-token supply changes (`hathorlib/headers/mint_melt_header.py`).

| Field | Size | Description |
|-------|------|-------------|
| Header ID | 1 B | `0x14` (mint) / `0x15` (melt) |
| `num_entries` | 1 B | `1 ≤ n ≤ 16` |
| Entries | variable | Concatenated entries |

Each entry (`MintMeltEntry`):

| Field | Size | Description |
|-------|------|-------------|
| `token_index` | 1 B | `1 ≤ token_index ≤ 16` (validated against `len(tx.tokens)` during verification; HTR/index 0 forbidden) |
| `amount` | 8 B BE | Public amount, `1 ≤ amount < 2^64` |

Deserialization rejects duplicate `token_index` within a header, out-of-range indices, and zero amounts.

### Wire Format

```
AmountShieldedOutput serialization:
  mode(1B=0x01) | commitment(33B) | rp_len(2B BE) | range_proof(var) |
  script_len(2B BE) | script(var) | token_data(1B) | ephemeral_pubkey(33B)

FullShieldedOutput serialization:
  mode(1B=0x02) | commitment(33B) | rp_len(2B BE) | range_proof(var) |
  script_len(2B BE) | script(var) | asset_commitment(33B) |
  sp_len(2B BE) | surjection_proof(var) | ephemeral_pubkey(33B)

ShieldedOutputsHeader:
  0x12 | num_outputs(1B) | <output_0> | <output_1> | ...

UnshieldBalanceHeader:
  0x13 | excess_blinding_factor(32B)

MintHeader / MeltHeader:
  0x14|0x15 | num_entries(1B) | (token_index(1B) | amount(8B BE)) ...

Single-entry MintHeader for 100,000 of token at index 1:
  0x14 0x01 0x01 0x00 0x00 0x00 0x00 0x00 0x01 0x86 0xA0
```

The `mode` byte discriminates output types during deserialization. Length fields are big-endian unsigned 16-bit integers (`!H`). `ephemeral_pubkey` is always written as 33 bytes; all-zeros encodes "not present".

### Sighash Coverage

| Structure | Included in sighash | Excluded |
|---|---|---|
| `AmountShieldedOutput` | mode, commitment, token_data, script, ephemeral_pubkey | range_proof |
| `FullShieldedOutput` | mode, commitment, asset_commitment, script, ephemeral_pubkey | range_proof, surjection_proof |
| `ShieldedOutputsHeader` | header_id, num_outputs, per-output sighash bytes | — |
| `UnshieldBalanceHeader` | full serialization | — |
| `MintHeader` / `MeltHeader` | full serialization | — |

Range and surjection proofs are excluded because they are verified independently and do not affect the spending signature; `ephemeral_pubkey` is always included to prevent a malleability attack that strips it.

## 4.3 Transaction Rules

The following rules govern shielded transactions. Each names the enforcing function in `hathor/verification/transaction_verifier.py` and the exception raised.

### Rule 1: Commitment / structure validity

`verify_commitments_valid` — every commitment, asset commitment, and (present) ephemeral pubkey is a valid 33-byte secp256k1 point, and the output count is within `MAX_SHIELDED_OUTPUTS`. Raises `InvalidShieldedOutputError`. Standard transaction structure rules (≥1 input, ≥1 output) continue to apply.

### Rule 2: Fee Is Always Transparent

`verify_shielded_fee` — a shielded transaction MUST carry a `FeeHeader`. Fees enter the balance equation as trivial commitments. Raises `InvalidShieldedOutputError`.

### Rule 3: Range Proofs

`verify_range_proofs` — every shielded output MUST include a valid range proof for `[1, 2^40)` against its commitment and generator (unblinded for `AmountShieldedOutput`, blinded for `FullShieldedOutput`). Raises `InvalidRangeProofError`.

### Rule 4: Trivial Commitment Protection

`verify_trivial_commitment_protection` — if a transaction has **any** shielded outputs, it MUST have **at least 2**. Raises `TrivialCommitmentError`.

> This is stricter than earlier drafts, which relaxed the rule when an input was
> shielded. The implementation enforces ≥2 unconditionally (a conservative,
> storage-free check) whenever shielded outputs are present.

### Rule 5: Homomorphic Balance

`verify_shielded_balance` — the balance equation of [§4.1](#41-cryptographic-primitives) holds (augmented per Rule M4 when mint/melt headers are present). Also enforces the mutual-exclusion invariants on `UnshieldBalanceHeader`. Raises `ShieldedBalanceMismatchError`.

### Rule 6: Surjection Proofs

`verify_surjection_proofs` — every `FullShieldedOutput` MUST include a valid surjection proof whose domain is `(transparent inputs ∪ shielded inputs ∪ MintHeader tokens)`. `AmountShieldedOutput` needs none (its token is visible via `token_data`). Raises `InvalidSurjectionProofError`.

### Rule 7: Authority Outputs Remain Transparent

`verify_authority_restriction` — no shielded output may carry authority bits. `AmountShieldedOutput` with `token_data & TOKEN_AUTHORITY_MASK` set is rejected; `FullShieldedOutput` has no `token_data` field and cannot encode authority. Raises `ShieldedAuthorityError`.

### Rule M1: Mint/Melt requires a shielded transaction

`verify_mint_melt_requires_shielded` — `MintHeader`/`MeltHeader` are valid only when the transaction carries a `ShieldedOutputsHeader` or an `UnshieldBalanceHeader`. Raises `ShieldedMintMeltForbiddenError`.

### Rule M2: Authority input required per entry

`verify_mint_melt_authority_inputs` — each `MintHeader` entry requires a matching transparent mint authority input; each `MeltHeader` entry requires a matching melt authority input. (A `TokenCreationTransaction` is exempt for `token_index = 1`, which it implicitly authorizes.) Raises `ForbiddenMint` / `ForbiddenMelt`.

### Rule M3: One direction per token

`verify_mint_melt_headers_well_formed` — a `token_index` appearing in `MintHeader` MUST NOT also appear in `MeltHeader`. Also bounds each `token_index` against `len(tx.tokens)`. Raises `InvalidMintMeltHeaderError`.

### Rule M4: Augmented homomorphic balance

`verify_shielded_balance` + `_fold_mint_melt_entry`. The balance equation becomes:

```
sum(C_in) + sum_T(mint_T · H_T) + withdraw · H_HTR
  == sum(C_out) + sum_T(melt_T · H_T) + deposit · H_HTR + fee · H_HTR + excess · G
```

- Each `MintHeader` entry `(T, amount)` adds `amount · H_T` to the **input** side (unblinded).
- Each `MeltHeader` entry `(T, amount)` adds `amount · H_T` to the **output** side (unblinded).
- For `DEPOSIT`-version tokens: mint adds a `deposit = 0.01 × amount` HTR term on the output side (paid by the user); melt adds a `withdraw = 0.01 × amount` HTR term on the input side (returned to the user). Computed via `get_deposit_token_deposit_amount` / `get_deposit_token_withdraw_amount`.
- For `FEE`-version tokens: each entry (mint *or* melt) adds one `FEE_PER_OUTPUT` HTR term on the output side. These per-entry charges are folded into the balance equation, **not** declared in `FeeHeader`.

Raises `ShieldedBalanceMismatchError`.

### Rule M5: No undeclared mint/melt

`verify_no_undeclared_mint_melt` — on a shielded transaction, any non-`NATIVE` token showing a transparent surplus (mint) or deficit (melt) in the token accounting MUST be covered by a corresponding `MintHeader`/`MeltHeader` entry. Without a public scalar, the prover could mint from nothing. Raises `ShieldedMintMeltForbiddenError`.

### Rule M6: Mint/Melt vs. NanoHeader

`verify_mint_melt_nano_compatibility` — a `NanoHeader` may coexist with mint/melt headers, but a single token MUST NOT have its supply changed through both a NanoHeader action and a `MintHeader`/`MeltHeader` entry (this would double-count in the balance equation). Raises `InvalidMintMeltHeaderError`.

### TokenCreationTransaction rules

`verify_minted_tokens` (token creation verifier): a shielded TCT MUST declare initial supply via exactly one `MintHeader` entry for `token_index = 1`, and MUST NOT melt that token. Non-shielded TCTs keep the existing `token_info.amount > 0` check. Raises `InvalidToken`. `MeltHeader` is not permitted on a token creation transaction.

## 4.4 Verification Pipeline

Verification follows Hathor's two-phase architecture.

### Phase 1: Without Storage

`verify_shielded_outputs` (no UTXO lookups):

```
verify_shielded_outputs(tx)
  |-- verify_commitments_valid            [Rule 1]
  |-- verify_authority_restriction        [Rule 7]
  |-- verify_range_proofs                 [Rule 3]
  |-- verify_trivial_commitment_protection[Rule 4]
  |-- verify_shielded_fee                 [Rule 2, lower bound]

verify_mint_melt_basic(tx)                (only if a mint/melt header is present)
  |-- verify_mint_melt_headers_well_formed[Rule M3 + bounds]
  |-- verify_mint_melt_requires_shielded  [Rule M1]
  |-- verify_mint_melt_nano_compatibility [Rule M6]
```

### Phase 2: With Storage

Requires UTXO lookups to resolve input types:

```
verify_shielded_outputs_with_storage(tx)
  |-- verify_surjection_proofs            [Rule 6]

(from _verify_tx, the shielded counterpart to transparent balance:)
  |-- verify_no_undeclared_mint_melt      [Rule M5]
  |-- verify_mint_melt_authority_inputs   [Rule M2]
  |-- verify_token_rules(shielded_fee=X)  [exact fee match]
  |-- verify_shielded_balance             [Rule 5 / Rule M4]
```

Shielded inputs are detected by index: a `TxInput` whose `index >= len(spent_tx.outputs)` references a shielded output at `index − len(spent_tx.outputs)` in the spent transaction's `ShieldedOutputsHeader`.

## 4.5 Mint/Melt Reference
[45-mintmelt-reference]: #45-mintmelt-reference

The mint/melt design lives entirely in the headers and the augmented balance equation; there is **no separate feature flag** — it is gated by `SHIELDED_TRANSACTIONS` along with the rest of shielded outputs.

### Disclosure model

Each `MintHeader`/`MeltHeader` entry discloses a **token reference** (`token_index`) and an **amount**, both in plaintext.

- **Token reference: always transparent.** The mint/melt authority output is itself transparent (Rule 7), so the token UID is already exposed; and the verifier must read the token's version (`NATIVE`/`DEPOSIT`/`FEE`) to apply the right deposit-or-fee logic.
- **Amount: transparent.** Total token supply is publicly computable per token by summing entries across the chain. Plaintext amounts are required for `DEPOSIT`-version tokens because the HTR deposit is `0.01 × amount` and must be verifiable. (Shielded amounts — a Pedersen commitment + range proof in the header — are a deferred business decision; see [Future possibilities](#future-possibilities).)

### Surjection-domain extension

For each `(T, amount)` in `MintHeader`, the unblinded NUMS asset tag `H_T = derive_asset_tag(T)` is appended to the surjection-proof domain so a `FullShieldedOutput` of a freshly-minted token can prove its asset is in `(transparent inputs ∪ shielded inputs ∪ minted tokens)`. `MeltHeader` does NOT extend the domain (melt produces no new outputs of the melted token).

### Header order

`MintHeader` (`0x14`) precedes `MeltHeader` (`0x15`) by the ascending-ID rule. Both follow `ShieldedOutputsHeader`/`UnshieldBalanceHeader`.

## 4.6 Fee Mechanism

### Settings

| Setting | Default | Location | Description |
|---------|---------|----------|-------------|
| `FEE_PER_AMOUNT_SHIELDED_OUTPUT` | 1 | `hathor/conf/settings.py` | HTR base units per `AmountShieldedOutput` |
| `FEE_PER_FULL_SHIELDED_OUTPUT` | 2 | `hathor/conf/settings.py` | HTR base units per `FullShieldedOutput` |
| `FEE_PER_OUTPUT` | 1 | `hathorlib` settings | HTR per chargeable transparent output and per `FEE`-token mint/melt entry |

### Calculation

```python
shielded_fee = (n_amount_shielded * FEE_PER_AMOUNT_SHIELDED_OUTPUT)
             + (n_full_shielded   * FEE_PER_FULL_SHIELDED_OUTPUT)
```

### Two-Phase Fee Verification

1. **Without storage (lower bound)**: `total_declared_fee >= shielded_fee` (`verify_shielded_fee`). The exact standard-token fee depends on chargeable outputs, which need UTXO lookups.
2. **With storage (exact match)**: `verify_token_rules(..., shielded_fee=X)` requires `standard_fee + shielded_fee == fees_from_fee_header`, and `verify_shielded_balance` reconciles the fee entries inside the balance equation. Both over- and under-payment are rejected.

Shielded outputs are not counted in `chargeable_outputs` for standard token fee math; to prevent fee avoidance (shielding an output to dodge `FEE_PER_OUTPUT`), the shielded fee rates are set at least as large as the standard per-output fee.

## 4.7 Network-Wide Supply Audit
[47-network-wide-supply-audit]: #47-network-wide-supply-audit

A monitor with chain access can verify, for each token, that no value was created or destroyed across the entire shielded history — **with no protocol change**:

```
Σ C_utxo  ==  Σ_token (total_supply_token · H_token)
```

This holds because the per-tx balance check is a curve-point equality (`Σ C_in == Σ C_out + fee·H_HTR`, augmented by the public mint/melt scalars), which by NUMS generator independence forces `Σ r_in = Σ r_out` for every shielded tx. Telescoping over the chain gives `Σ r_utxo = 0`, collapsing the residual `G`-component to zero. `total_supply_token` is maintained from the public mint/melt declarations (and genesis allocation for HTR).

A regression test (`hathor_tests/tx/test_shielded_audit_equation.py`) builds small chains using `htr_lib.shielded` directly and checks the equation; the full derivation is in the design memo at `_designs/03-amount-privacy/100-NETWORK-SUPPLY-AUDIT.md`. Inflation-bug monitoring can therefore ship against the current chain immediately.

## 4.8 ECDH Recovery Mechanism

### Sender Flow

1. Generate ephemeral keypair `(e, E = e*G)` on secp256k1 (`generate_ephemeral_keypair`).
2. Obtain the recipient's scan public key `P = B_scan`.
3. Compute shared secret `s = ECDH(e, P)` (secp256k1 `SharedSecret`, i.e. SHA256 over the shared point).
4. Derive deterministic nonce `nonce = SHA256("Hathor_CT_nonce_v1" || s)` (re-hashed with a counter in the ~2⁻¹²⁸ case where the result is not a valid scalar).
5. Create the range proof using `nonce` as the proof's nonce key (not random), enabling rewind.
6. For `FullShieldedOutput`: embed `token_uid(32B) || asset_blinding_factor(32B)` (64-byte message) in the range proof.
7. Store `E` (33 bytes, compressed) in the output's `ephemeral_pubkey` field.

### Recipient Flow

1. Match the output script against a wallet key.
2. Extract `E`; compute `s = ECDH(scan_privkey, E)` (equal to the sender's `s`).
3. Derive `nonce = SHA256("Hathor_CT_nonce_v1" || s)`.
4. Call `rewind_range_proof(proof, commitment, nonce, generator)` with `generator = derive_asset_tag(token_uid)` for `AmountShieldedOutput` or `output.asset_commitment` for `FullShieldedOutput`.
5. Receive `(value, blinding_factor, message)`.
6. For `FullShieldedOutput`: extract `token_uid` and `asset_blinding_factor` from `message`, recompute the expected asset commitment, and cross-check it against the on-chain `asset_commitment` (rejects a fraudulent token UID).

### Security Properties

- **Sighash binding**: `ephemeral_pubkey` is in the sighash, preventing MITM replacement.
- **Failed rewind**: returns an error (not garbage) on the wrong nonce — no false positives.
- **Forward secrecy / nonce uniqueness**: ephemeral keys are single-use.
- **Domain separation**: `"Hathor_CT_nonce_v1"` isolates this derivation; `"Hathor_AssetTag_v1"` isolates asset-tag derivation.
- **Token UID cross-check**: for `FullShieldedOutput`, the recovered UID is bound to the `asset_commitment`.

## 4.9 Feature Activation

```python
# hathor/feature_activation/feature.py
class Feature(StrEnum):
    SHIELDED_TRANSACTIONS = 'SHIELDED_TRANSACTIONS'
```

```python
# hathor/conf/settings.py
ENABLE_SHIELDED_TRANSACTIONS: FeatureSetting = FeatureSetting.DISABLED
```

`FeatureSetting` is an enum with values `DISABLED`, `ENABLED`, `FEATURE_ACTIVATION`. A single flag gates **all** shielded functionality, including mint/melt headers — there is no separate `SHIELDED_MINT_MELT` flag. The vertex parser only parses shielded headers when the flag is active, and `get_maximum_number_of_headers()` is raised from 3 to 5 under the same gate.

The native `htr_lib` crypto extension must be available for shielded operations to function; it is imported by the `hathorlib` wrappers at runtime.

## 4.10 Interaction with Other Features

### Silent Payments (Phase A)

The `ephemeral_pubkey` serves double duty: without SP it derives blinding factors only (the recipient address is visible in the script); with SP the same ECDH shared secret derives both a one-time recipient address and the blinding factors. A single ECDH operation then provides recipient privacy + blinding-factor communication.

### Ring Signatures / Nullifiers (Phase C)

All shielded outputs are opaque commitments, so any shielded output is a valid decoy regardless of hidden amount or token — larger anonymity sets with simpler selection. Surjection proofs compose naturally: the domain can include ring members (real + decoys).

### Nano Contracts

A `NanoHeader` may coexist with mint/melt headers, subject to Rule M6 (no same-token supply change via both channels). Deeper interaction — contracts consuming or producing shielded outputs — is out of scope for this RFC.

# Drawbacks
[drawbacks]: #drawbacks

1. **Transaction size increase.** A shielded output is ~3.3 KB (dominated by the ~3213-byte Borromean range proof) vs ~40 bytes for a transparent output — roughly two orders of magnitude larger. This increases storage, bandwidth, and propagation time. The range proof is the single largest contributor; a Bulletproof-based proof (~675 B) would shrink this substantially but is not available in the current `secp256k1-zkp` build.

2. **Verification cost.** Each range proof and each surjection proof is verified independently. Batch verification is not yet truly batched (the helper loops sequentially), so multi-output transactions pay close to linear cost.

3. **Wallet complexity.** Wallets must implement ECDH key exchange, range-proof rewind, blinding-factor management and balancing, surjection-proof generation, and fee computation — a substantial increase over transparent-only transactions.

4. **Explorer capability reduction.** Explorers lose amount/balance display for shielded outputs (though per-token supply remains auditable from the mint/melt headers).

5. **Recipient public key requirement.** The sender must know the recipient's full compressed public key (not just the address hash) to establish the ECDH shared secret. This requires a prior on-chain spend, a payment protocol, or a new address format.

6. **Full-unshield scalar leak.** `UnshieldBalanceHeader` exposes `Σ r_in` as a raw scalar. A point-form replacement with a binding signature would close this but is not implemented (see Future possibilities).

7. **40-bit value ceiling.** Range proofs prove `[1, 2^40)`. This is above the maximum HTR supply but is a smaller domain than the full `u64` amount space; any token whose supply could exceed `2^40 − 1` base units cannot be fully represented in a shielded output.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why header-based (not a new TxVersion)?

A new `TxVersion` would require changes throughout the codebase — every `match vertex.version:`, serialization path, feature gate, and verification path. The header-based approach reuses Hathor's existing header infrastructure (already used by `NanoHeader` and `FeeHeader`), attaching shielded data to standard transactions with zero structural changes, so existing mempool logic and DAG management continue to work unmodified.

## Why two output types (not one)?

`AmountShieldedOutput` (amount hidden, token visible) skips the surjection proof and so is smaller and cheaper to verify than `FullShieldedOutput`. Many use cases only need amount privacy (e.g. salaries paid in HTR), so offering both tiers lets users pay only for the privacy they need.

## Why separate range proofs (not aggregated)?

Aggregated proofs would require the prover to know all values and blinding factors for every output, which is incompatible with atomic swaps and multi-party transactions where each party independently constructs their outputs. Separate proofs also avoid permanently linking all outputs as same-party and allow independent UTXO pruning.

## Why Borromean range proofs (not Bulletproofs)?

The implementation uses `secp256k1-zkp`'s `RangeProof` (a Borromean-style proof). It is the proof type exposed by the library version in use, battle-tested in production (Liquid/Elements), and supports the ECDH-rewind recovery mechanism directly. The cost is size: ~3213 bytes vs a Bulletproof's ~675 bytes. Bit width is pinned to 40 so all proofs are constant-size, removing a value-size side-channel. Migrating to Bulletproofs is a future optimization gated on library support.

## Why secp256k1-zkp (not custom crypto)?

`secp256k1-zkp` is the industry-standard library for Confidential Transactions, maintained by Blockstream and used by Liquid/Elements. It provides battle-tested Pedersen commitments, range proofs, and surjection proofs on the same secp256k1 curve Hathor already uses. Writing custom cryptographic primitives would be a security liability.

## Why ECDH rewind (not encrypted messages)?

Range-proof rewinding is an established technique (Monero, Grin, Elements) that embeds recovery data inside the proof itself, adding zero bytes to the transaction. Encrypting amount/blinding data in a separate field would increase size and add a new encryption scheme to audit. Rewinding also authenticates: only the correct ECDH shared secret produces valid rewind results.

## Why declare mint/melt amounts publicly?

Pedersen commitments cannot, on their own, distinguish "minted from nothing" from "received from an input". A public scalar declaring the mint binds it into the balance equation, preserving the no-inflation guarantee and keeping supply auditable. A purely private alternative (a ZK "I am minting Y ≤ my authority limit" proof) is an order of magnitude more complex and loses auditability.

## Why a scalar excess (not a binding signature) today?

The implemented full-unshield path publishes the residual as a scalar in `UnshieldBalanceHeader`. The scalar form is functional and prevents inflation (the verifier checks balance directly against it). A point-form excess plus a Schnorr binding signature (Sapling-style) would additionally hide `Σ r_in` and enable receiver-chosen blindings and non-interactive multi-party construction — but those capabilities are not yet required, so the simpler scalar form ships first. See Future possibilities.

## Alternative: ZK-SNARKs (Zcash model)

Zcash's Sapling/Orchard circuits hide amounts, token types, and the sender in one proof. This requires a trusted setup (Groth16) or larger proofs (Halo 2 ~1.8 KB), a different curve (BLS12-381 or Pallas/Vesta) incompatible with secp256k1, complex circuit auditing, and far slower proving. Hathor's phased approach reaches a similar end-state with individually simpler, more auditable components.

## Alternative: Mimblewimble

Mimblewimble (Grin, Beam, Litecoin MWEB) provides amount hiding with cut-through, but requires interactive transaction construction, fundamentally changes the UTXO model, cannot support scripting or multi-asset transactions in its current form, and is incompatible with Hathor's DAG structure.

## Impact of not doing this

Without shielded outputs, all Hathor transactions remain fully transparent, with no on-chain option for amount or token privacy — limiting Hathor's utility for business payments, DeFi, and personal finance, while competitors with privacy features hold a structural advantage.

# Prior art
[prior-art]: #prior-art

## Liquid / Elements Confidential Assets

The primary inspiration. Liquid implements Confidential Transactions with Pedersen commitments, range proofs, and asset surjection proofs on secp256k1. Hathor uses the same `secp256k1-zkp` library and constructions. Liquid also supports confidential asset issuance with a public asset_id and amount — directly analogous to `MintHeader`. The two-output-type split (amount-only vs. full) is a Hathor addition.

## Monero (RingCT)

Monero uses Pedersen commitments and range proofs for amount hiding plus ring signatures for sender privacy; all transactions are mandatory CT. Differences: Ed25519 (not secp256k1), mandatory (not opt-in), combined amount+sender privacy, no custom tokens. Lessons: range-proof CT is production-proven at scale; ECDH recovery with deterministic nonces is the standard approach.

## Zcash (Sapling / Orchard)

Zcash uses ZK-SNARKs for full transaction privacy in a single proof, on BLS12-381 / Pallas-Vesta. Sapling's `valueBalance` field and binding signature are the closest analogs to a per-asset public delta on an otherwise-shielded transaction (the inspiration for the deferred point-form excess). Differences: different curves, more expensive proving, separate shielded pool. Lesson: opt-in privacy reduces the anonymity set, so shielded usage should be encouraged.

## Grin / Beam (Mimblewimble) and Litecoin MWEB

Mimblewimble provides amount hiding with cut-through; MWEB adds opt-in CT via extension blocks with ECDH stealth addresses. Not adopted because of interactive construction, incompatibility with scripting/multi-asset, and incompatibility with Hathor's DAG. Mimblewimble's per-tx "kernel excess" (a point + Schnorr signature) is structurally the binding signature discussed under Future possibilities.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Shielded address format.** Option A (compact) vs Option B (full keys) vs Option C (spend-only). The tradeoff is address length vs. Silent Payments compatibility and light-client scanning. Should be resolved before wallet implementations begin. (Code currently assumes the full public key — Option B.)

2. **Final fee amounts.** The placeholders (`FEE_PER_AMOUNT_SHIELDED_OUTPUT = 1`, `FEE_PER_FULL_SHIELDED_OUTPUT = 2`) are not final and should be set from measured verification-cost differentials and network economics.

3. **Batch verification.** The current `batch_verify_range_proofs` is a sequential loop. A true batched multi-exponentiation API would reduce CPU for multi-output transactions; timeline depends on library support.

4. **Light client scanning.** Full nodes verify all proofs, but light clients need an efficient protocol for detecting their own shielded payments without downloading every transaction (SP-style filters, server-side scanning, or compact summaries).

5. **Shielded Nano Contract interactions.** Beyond the Rule M6 same-token guard, can a contract consume shielded inputs or produce shielded outputs? Deferred to a future RFC.

6. **`get_maximum_number_of_headers()` value.** 5 is the current value; whether to leave more margin for future headers is a consensus parameter decision.

7. **40-bit range bound.** Whether `RANGE_PROOF_BITS = 40` is the right ceiling for all current and future tokens, or whether a per-token bit width is warranted.

# Future possibilities
[future-possibilities]: #future-possibilities

## Binding signature / point-form excess (designed, not implemented)

Two related upgrades share one primitive — a per-tx Schnorr "binding signature" over a Pedersen value-balance commitment, with the kernel-excess point `E_tx = e_tx · G` as the public key:

- **Point-form `UnshieldBalanceHeader`.** Replace the 32-byte scalar `excess_blinding_factor` with `E_tx = excess · G` (33 B) plus a Schnorr signature (`R` 33 B, `z` 32 B). This keeps `Σ r_in` behind a discrete-log barrier (closing the full-unshield scalar leak) while the signature preserves inflation prevention. Cost: ~+66 B per full-unshield transaction; affects no other shape. *Recommended as the first follow-up.*

- **Binding signature for every shielded transaction.** Extend `ShieldedOutputsHeader` with the same `(E_tx, R, z)` trailer, removing the "last output absorbs the residual" construction. This enables independent per-output blindings, receiver-chosen (Sapling-style) blindings, composable output addition (no range-proof recomputation when adding an output), and non-interactive multi-party construction. Cost: ~+98 B per shielded transaction plus a verifier change. Justified only when a concrete trigger lands: Sapling-style payment proofs, hardware-wallet cold-signer flows, shielded coin-join / atomic swaps, or measured output-addition latency.

The construction is exactly Sapling's binding signature (also used by Mimblewimble and Liquid CT): deterministic Schnorr over secp256k1 with a `"HathorBindingSig/v1"` domain separator, batch-verifiable across a block. Neither variant is in the current code; the network-wide supply audit ([§4.7](#47-network-wide-supply-audit)) already works without it.

## Bulletproof range proofs

Migrating from Borromean (~3213 B) to Bulletproof (~675 B) range proofs would cut shielded-output size ~5× and enable true batch verification, gated on `secp256k1-zkp` exposing a compatible Bulletproof + rewind API.

## Shielded mint/melt amounts

Extend `MintHeader`/`MeltHeader` to optionally carry a Pedersen commitment plus a range proof (and, for `DEPOSIT`-version tokens, a proof binding the HTR deposit to 1% of the committed amount), hiding total supply at the cost of the public-supply-auditor property. A business decision deferred to a later RFC; retrofittable via a new feature flag without invalidating already-issued tokens.

## Aggregated Bulletproofs for single-party transactions

When all outputs are controlled by one party (the common non-swap case), a single aggregated proof could reduce total proof size logarithmically, if an MPC-free aggregation path becomes available.

## View key delegation for compliance

A scan-key-derived view key would let a designated party (auditor, regulator, exchange) decrypt shielded outputs addressed to them without spending authority — analogous to Monero view keys.

## Asset-balance binding signature

The (future) binding signature covers value balance; a second one over asset balance would enable chain-wide *asset* audit for `FullShieldedOutput` transactions, detecting cross-asset forgery the same way inflation is detected today.

## Tiered privacy fees

As the privacy stack matures, fees could be tiered by privacy level (transparent < amount-shielded < full-shielded < ring-shielded), extending the per-output-type fee model established here.
