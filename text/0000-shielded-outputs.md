- Feature Name: shielded_outputs
- Start Date: 2026-03-03
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Hathor Labs

# Summary
[summary]: #summary

This RFC introduces **shielded outputs** for Hathor transactions — a header-based extension that hides output amounts and, optionally, token types using Pedersen commitments, Bulletproof range proofs, and asset surjection proofs. Rather than defining a new transaction version, shielded outputs attach to existing transaction types (regular transactions and token creation transactions) via a `ShieldedOutputsHeader`, preserving full backward compatibility. Two privacy tiers are offered: `AmountShieldedOutput` (amount hidden, token visible) and `FullShieldedOutput` (both amount and token hidden). Recipients recover hidden values through ECDH-based range proof rewinding, requiring no out-of-band communication.

# Motivation
[motivation]: #motivation

**Privacy is a fundamental property of money.** Physical cash does not broadcast the denomination of every bill exchanged between two parties, yet transparent blockchains do exactly that. Every Hathor transaction today reveals the exact amount transferred, the token involved, and — combined with address analysis — a detailed picture of economic activity.

This transparency creates real problems:

- **Payroll and compensation.** An employer paying salaries on-chain reveals every employee's compensation to anyone who looks.
- **Business-to-business payments.** Suppliers, vendors, and partners can reverse-engineer pricing, margins, and deal terms from on-chain flows.
- **Trading and DeFi.** Front-runners and MEV extractors exploit visible amounts to sandwich trades or copy strategies.
- **Personal finance.** Any recipient of a payment can trace the sender's full balance and transaction history.
- **Multi-token privacy.** Hathor supports custom tokens. When token types are visible, observers can track the flow of specific assets (e.g., loyalty points, governance tokens, stablecoins), revealing business relationships and portfolio composition.

Shielded outputs address these problems by making amounts and token types cryptographically opaque to everyone except the transaction participants, while preserving the ability of every full node to verify that no inflation or double-spending has occurred.

**Expected outcome:** Users can opt into amount privacy (and optionally token-type privacy) on a per-output basis, within ordinary Hathor transactions, with no protocol-level changes to transaction versions, input formats, or the DAG structure.

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
| `AmountShieldedOutput` | Yes | No | Range proof (~675 B) | Hide salary amounts while token type is public |
| `FullShieldedOutput` | Yes | Yes | Range proof + surjection proof (~675 + ~130 B) | Full privacy for multi-token transactions |

Both tiers use **Pedersen commitments** for amounts and **Bulletproof range proofs** to guarantee amounts are non-negative. `FullShieldedOutput` additionally uses **blinded asset tags** and **asset surjection proofs** to hide and validate the token type.

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
|  Pedersen commitments + Bulletproofs + surjection proofs  |
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

Note: at least 2 shielded outputs are required when all inputs are transparent (Rule 4 — see Section 4.3). This prevents an observer from trivially deducing the single output's amount.

### Unshielding: Returning to Transparency

To unshield, a user spends shielded inputs into transparent outputs:

```
UNSHIELDING TRANSACTION:
  Input:   [commitment] (shielded, 90 HTR hidden)
  Output0: 60 HTR to Bob (transparent)
  Output1: [commitment'] (shielded change, 25 HTR hidden)
  Fee:     5 HTR (transparent)

Observer sees: 60 HTR left the shielded pool, plus a shielded change output
Observer does NOT know: the shielded input amount or the change amount
```

### Mixed Transactions

Transparent and shielded inputs/outputs can be freely combined in a single transaction. The balance equation works uniformly: transparent amounts are treated as "trivial commitments" with a zero blinding factor, and the homomorphic verification covers everything in one check.

| Type | Transparent In | Shielded In | Transparent Out | Shielded Out | Use Case |
|------|:---:|:---:|:---:|:---:|---|
| Standard | Yes | — | Yes | — | Legacy transaction |
| Shielding | Yes | — | Optional | Yes (>=2) | Enter shielded pool |
| Confidential | — | Yes | — | Yes | Fully private transfer |
| Unshielding | — | Yes | Yes | Optional | Exit shielded pool |
| Fully mixed | Yes | Yes | Yes | Yes | Maximum flexibility |

### Fee Model

Shielded outputs impose additional verification costs (Bulletproof verification ~1ms per output, surjection proof verification, larger storage). A per-output fee compensates the network:

- `FEE_PER_AMOUNT_SHIELDED_OUTPUT`: charged per `AmountShieldedOutput` (default: 1 HTR base unit)
- `FEE_PER_FULL_SHIELDED_OUTPUT`: charged per `FullShieldedOutput` (default: 2 HTR base units)

Fees are declared in the existing `FeeHeader`, are fully transparent, and are burned. See [Section 4.5](#45-fee-mechanism) for details.

### What Remains Visible

Even with shielded outputs, the following information is always public:

- **Transaction structure**: number of inputs, number of outputs, which outputs are shielded vs. transparent
- **Fee amounts**: always plaintext HTR (Rule 2)
- **Authority outputs**: mint/melt authority tokens are always transparent (Rule 7)
- **Scripts**: the locking script (recipient address) remains visible unless combined with Silent Payments (Phase A)
- **Transparent inputs/outputs**: their amounts and tokens remain fully visible

## 3. Shielded Addresses (Pending Decision)

Receiving shielded outputs requires the sender to know the recipient's full public key (not just the address hash) in order to establish the ECDH shared secret for blinding factor communication. Today, P2PKH addresses only encode a hash of the public key.

A new **shielded address type** would bundle the necessary keys into a single address the recipient can publish.

### Key Components

A shielded address encodes two keys:

- **Scan public key** (`B_scan`, 33 bytes): Used for ECDH — the sender computes a shared secret with this key, and the recipient uses the corresponding private key to recover blinding factors.
- **Spend public key** (`B_spend`, 33 bytes): Controls spending authority. The output script locks funds to this key.

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

Only the spend public key is encoded — no scan key at all. This is the shortest possible shielded address and the simplest to implement. However, without a scan key, the wallet cannot delegate blockchain scanning to a third-party service (since the scan private key is what enables a service to detect incoming payments without having spending authority). The wallet must download and trial-decrypt the entire shielded transaction history itself to compute its balance, which is impractical for light clients.

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

**Status:** Decision pending. Both options are documented here for review. The implementation currently uses the full public key for ECDH, so Option B aligns with the existing code path.

## 4. Wallet Integration Guide

### Receiving Shielded Payments

Every shielded output contains an `ephemeral_pubkey` field (33 bytes). The recipient recovers the hidden amount through **ECDH + range proof rewind**:

1. **Address match**: The wallet checks if the output script pays to one of its addresses.
2. **ECDH shared secret**: `s = SHA256(privkey * ephemeral_pubkey)`.
3. **Nonce derivation**: `nonce = SHA256("Hathor_CT_nonce_v1" || s)`.
4. **Range proof rewind**: Using the nonce as the rewind key, the wallet calls `rewind_range_proof()` which returns the committed `(value, blinding_factor, message)`.
5. **For `FullShieldedOutput`**: The `message` field contains `token_uid(32B) || asset_blinding_factor(32B)`. The wallet reconstructs the expected asset commitment from these values and cross-checks against the on-chain `asset_commitment` to prevent a malicious sender from claiming a worthless token is HTR.

If rewind fails (wrong nonce), the output is not addressed to this wallet — skip it silently.

### Sending Shielded Payments

1. **Choose privacy tier** per output (`AmountShieldedOutput` or `FullShieldedOutput`).
2. **Generate ephemeral keypair** `(e, E = e*G)` for each output.
3. **Compute ECDH shared secret** with the recipient's public key.
4. **Derive nonce** for range proof construction.
5. **Create Pedersen commitment**: `C = amount * H_token + blinding * G`.
6. **Create range proof** using the deterministic nonce (enables recipient rewind).
7. **For `FullShieldedOutput`**: Create blinded asset commitment and surjection proof.
8. **Balance blinding factors**: Assign random blinding factors to all shielded outputs except the last, which receives the balancing factor: `r_last = sum(r_inputs) - sum(r_other_outputs)`.
9. **Compute and attach fee** in `FeeHeader`.

### Blinding Factor Management and Backup

The wallet must store blinding factors for every owned shielded UTXO — without them, the funds cannot be spent.

**Recovery guarantee**: Since blinding factors for received outputs are derived deterministically from `ECDH(wallet_privkey, ephemeral_pubkey)` — and both the private key (from the seed) and the ephemeral pubkey (on-chain) are always available — a **seed backup is sufficient** for full recovery. The wallet re-derives all blinding factors by scanning the blockchain.

For change outputs, wallets should use deterministic derivation from the input blinding factors and output index to ensure recoverability.

### Balance Display

From the wallet owner's perspective, balances are fully accurate:

- **Total balance**: sum of transparent UTXOs + sum of decrypted shielded UTXOs.
- **Per-token breakdown**: the wallet knows which token each shielded UTXO holds.
- **Shielded vs. transparent breakdown**: optionally shown per token.

Privacy only affects external observers — the wallet owner always sees exact amounts.

### Rule 4 Compliance

When all inputs are transparent, the wallet must automatically ensure at least 2 shielded outputs (or include a transparent output). This is typically satisfied naturally (payment + change), but the wallet may need to create a zero-value absorber output in edge cases.

### Fee Computation

```
shielded_fee = (count_amount_shielded * FEE_PER_AMOUNT_SHIELDED_OUTPUT)
             + (count_full_shielded   * FEE_PER_FULL_SHIELDED_OUTPUT)
total_fee    = shielded_fee + standard_fee_if_any
```

The wallet should display the expected fee before the user confirms.

## 5. Explorer and Indexer Impact

### What Breaks

| Capability | Status | Reason |
|---|---|---|
| Show output amounts | Broken for shielded | Hidden behind Pedersen commitments |
| Identify output token type | Broken for `FullShieldedOutput` | Hidden behind blinded asset commitments |
| Compute address balances | Broken for shielded | Cannot sum opaque curve points |
| Token supply tracking | Unaffected | Mint/melt transactions cannot contain shielded outputs (Rule 8) |
| Rich list / rankings | Broken for shielded | Balances not computable |

### What Still Works

- Transaction structure (input/output count, shielded vs. transparent)
- Transparent outputs (amounts, tokens, scripts — unchanged)
- Fee amounts (always transparent)
- Cryptographic proof verification (anyone can verify correctness)
- Authority UTXO tracking (always transparent, Rule 7)
- Token metadata (name, symbol from creation transaction)

### Mitigations

- **Shielded pool boundary tracking**: Explorers can track aggregate amounts entering/leaving the shielded pool from the transparent side of shielding/unshielding transactions.
- **View key delegation**: Users can optionally share view keys with trusted explorers for selective disclosure.
- **"Shielded" indicator**: Explorers should display shielded outputs with a clear indicator, showing commitment hex values but never placeholder amounts.

## 6. Mint/Melt Transactions Are Always Transparent

Transactions that exercise mint or melt authority **cannot** contain shielded outputs (Rule 8). Both the authority outputs (Rule 7) and all value outputs in a mint/melt transaction remain fully transparent.

This means:

- An explorer can see **when** mint authority is exercised and **exactly how many** tokens were minted or melted.
- Token supply remains fully auditable for all custom tokens — no trust in the token creator is required.
- Users who want amount privacy must move minted tokens into shielded outputs in a separate transaction.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 4.1 Cryptographic Primitives

### Pedersen Commitments

A Pedersen commitment to a value `v` with blinding factor `r` using generator `H` is:

```
C = v * H + r * G

Where:
  v  = amount (u64, range [0, 2^64))
  r  = blinding factor (32-byte scalar)
  H  = generator point specific to the token type
  G  = secp256k1 base generator
```

**Properties:**
- **Hiding**: Given `C`, an observer cannot determine `v` without knowing `r`.
- **Binding**: Given `C`, one cannot find `(v', r')` such that `C = v'*H + r'*G` and `v' != v`, unless one knows the discrete log of `H` w.r.t. `G` (computationally infeasible for NUMS generators).
- **Homomorphic**: `C1 + C2 = (v1+v2)*H + (r1+r2)*G`. This enables balance verification without revealing amounts.

### NUMS Asset Tag Derivation

Each token has a deterministic generator `H_token` derived via a Nothing-Up-My-Sleeve (NUMS) construction:

```
H_token = NUMS_hash(token_uid)

Algorithm:
  tag = SHA256("Hathor_AssetTag_v1" || token_uid)
  H_token = generator_from_tag(tag)
```

The domain separator `"Hathor_AssetTag_v1"` prevents cross-protocol collisions. The construction guarantees no one knows `x` such that `H_token = x*G`, which is essential for the binding property.

**Token UID normalization**: HTR uses `token_uid = b'\x00'` (1 byte) internally, but the crypto library requires 32 bytes. The normalization function pads HTR's token UID with 31 zero bytes.

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

### Bulletproof Range Proofs

Each shielded output includes a Bulletproof range proof demonstrating:

```
The committed amount v satisfies: 1 <= v < 2^64
```

The lower bound of 1 (not 0) prevents zero-amount outputs that could be used in certain attack vectors.

**Architecture: separate proofs, not aggregated.** Each output carries its own independent Bulletproof. This design supports:

- **Multi-party transactions**: Each party generates proofs for their own outputs independently, without revealing amounts.
- **Atomic swaps**: No need to share blinding factors across parties.
- **UTXO pruning**: Spent output proofs can be discarded independently.

Performance optimization is achieved through **batch verification** (`verify_multi`), which amortizes multi-exponentiation cost across proofs (estimated 30-50% CPU reduction for multi-output transactions), and parallel verification across transactions.

**Proof size**: ~675 bytes typical, bounded by `MAX_RANGE_PROOF_SIZE = 1024` bytes.

### Asset Surjection Proofs

For `FullShieldedOutput` only. Proves the output's blinded asset commitment corresponds to one of the input asset commitments, without revealing which one.

```
Given:
  Input asset commitments:   A_1, A_2, ..., A_n
  Output asset commitment:   A_out

Compute differences: d_i = A_out - A_i for each input i

For the matching input j (same token):
  d_j = (H_token + r_out*G) - (H_token + r_j*G) = (r_out - r_j) * G
  -> discrete log is KNOWN

For non-matching inputs i != j (different token):
  d_i = (H_out + r_out*G) - (H_i + r_i*G) = (H_out - H_i) + (r_out - r_i)*G
  -> discrete log is UNKNOWN

A ring signature on {d_1, ..., d_n} proves knowledge of the discrete log
for exactly one d_i, without revealing which one.
```

**Proof size**: Grows linearly with the number of inputs in the surjection domain. For a typical transaction with 3 inputs: ~130 bytes. Maximum: `MAX_SURJECTION_PROOF_SIZE = 4096` bytes.

### Homomorphic Balance Verification

The balance equation covers all inputs and outputs uniformly:

```
sum(C_inputs) == sum(C_outputs) + sum(C_fee_entries)

Where:
  - Shielded inputs/outputs use their on-chain commitment directly
  - Transparent inputs/outputs use trivial commitments: C = amount * H_token
  - Fee entries from FeeHeader are treated as transparent outputs
```

Expanding the equation and grouping by generator:

```
(sum(v_in) - sum(v_out) - sum(fees)) * H  +  (sum(r_in) - sum(r_out)) * G  ==  O
```

Since `H` and `G` are linearly independent (no known discrete log relationship), both scalar coefficients must be zero:

1. `sum(v_in) = sum(v_out) + sum(fees)` — values balance.
2. `sum(r_in) = sum(r_out)` — blinding factors balance.

The wallet enforces condition (2) by construction: it assigns random blinding factors to all outputs except the last, which receives the balancing residual.

## 4.2 Data Structures

### AmountShieldedOutput

Hides the amount; token type remains visible.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `commitment` | bytes | 33 B | Pedersen commitment `C = v*H_token + r*G` |
| `range_proof` | bytes | ~675 B (max 1024) | Bulletproof range proof |
| `script` | bytes | variable (max 1024) | Locking script (P2PKH, etc.) |
| `token_data` | int | 1 B | Token index (same semantics as `TxOutput.token_data`) |
| `ephemeral_pubkey` | bytes | 33 B | Compressed secp256k1 point for ECDH recovery |

**Typical total size**: ~770 bytes per output (with P2PKH script).

### FullShieldedOutput

Hides both amount and token type.

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `commitment` | bytes | 33 B | Pedersen commitment `C = v*A + r*G` (uses blinded generator) |
| `range_proof` | bytes | ~675 B (max 1024) | Bulletproof range proof |
| `script` | bytes | variable (max 1024) | Locking script |
| `asset_commitment` | bytes | 33 B | Blinded asset tag `A = H_token + r_asset*G` |
| `surjection_proof` | bytes | ~130 B (max 4096) | Asset surjection proof |
| `ephemeral_pubkey` | bytes | 33 B | Compressed secp256k1 point for ECDH recovery |

**Typical total size**: ~930 bytes per output (with P2PKH script, 3-input surjection domain).

### ShieldedOutputsHeader

Shielded outputs are carried in a transaction header, not in the standard `tx.outputs` list.

| Field | Size | Description |
|-------|------|-------------|
| Header ID | 1 B | `0x12` (`VertexHeaderId.SHIELDED_OUTPUTS_HEADER`) |
| `num_outputs` | 1 B | Number of shielded outputs (max 32) |
| Outputs | variable | Concatenated serialized outputs |

**Maximum shielded outputs per transaction**: `MAX_SHIELDED_OUTPUTS = 32`.

### Wire Format

```
AmountShieldedOutput serialization:
  mode(1B=0x01) | commitment(33B) | rp_len(2B BE) | range_proof(var) |
  script_len(2B BE) | script(var) | token_data(1B) | ephemeral_pubkey(33B)

FullShieldedOutput serialization:
  mode(1B=0x02) | commitment(33B) | rp_len(2B BE) | range_proof(var) |
  script_len(2B BE) | script(var) | asset_commitment(33B) |
  sp_len(2B BE) | surjection_proof(var) | ephemeral_pubkey(33B)
```

The `mode` byte discriminates output types during deserialization. Length fields use big-endian unsigned 16-bit integers (`!H` struct format).

### Sighash Coverage

The transaction sighash includes: `mode`, `commitment`, `script`, `token_data` (amount-shielded) or `asset_commitment` (full-shielded), and `ephemeral_pubkey`. It **excludes** `range_proof` and `surjection_proof` — these are verified independently and do not affect the spending signature.

## 4.3 Transaction Rules

Seven rules govern shielded transactions:

### Rule 1: Minimum Structure

At least one input (or Nano Contract withdraw) and at least one output (transparent or shielded, or Nano Contract deposit) required. Standard transaction structure rules apply.

### Rule 2: Fee Is Always Transparent

The transaction fee is always a plaintext HTR amount, declared in a `FeeHeader`. The fee enters the balance equation as a trivial commitment: `C_fee = fee * H_HTR`.

### Rule 3: Blinding Factors Must Balance

```
sum(r_input_shielded) = sum(r_output_shielded)
```

Transparent inputs and outputs contribute `r = 0`. The wallet enforces this by choosing the last shielded output's blinding factor as the balancing residual. For `FullShieldedOutput`, asset blinding factors must also balance: `sum(s_inputs) = sum(s_outputs)`.

### Rule 4: Trivial Commitment Protection

If **all** inputs are transparent, at least 2 shielded outputs are required.

**Rationale**: With all transparent inputs, the total input amount is public. A single shielded output with no transparent outputs would have its blinding factor forced to zero (to satisfy Rule 3), making the commitment trivially deducible as `C = (total_input - fee) * H`.

**Exception**: This rule is relaxed if any input is shielded (the input's non-zero blinding factor provides the necessary entropy). It also does not apply if there is a transparent output alongside the single shielded output.

### Rule 5: Range Proofs

Every shielded output MUST include a valid Bulletproof range proof proving the committed amount is in `[1, 2^64)`. The minimum value of 1 (not 0) prevents zero-amount outputs.

### Rule 6: Surjection Proofs

Every `FullShieldedOutput` MUST include a valid asset surjection proof proving its asset commitment corresponds to one of the input asset commitments. `AmountShieldedOutput` does not require a surjection proof (its token is visible via `token_data`). Transparent inputs contribute their unblinded NUMS asset tag to the surjection proof domain.

### Rule 7: Authority Outputs Remain Transparent

Mint and melt authority outputs MUST always be transparent `TxOutput`s. Attempting to set authority bits on a shielded output is invalid (`ShieldedAuthorityError`). Authority tokens control token supply and must remain auditable.

### Rule 8: Mint/Melt Transactions Cannot Have Shielded Outputs

A transaction that contains any mint or melt operation (i.e., spends a mint or melt authority input) MUST NOT include any shielded outputs (`ShieldedMintMeltForbiddenError`). All value outputs in a mint/melt transaction must be transparent.

**Rationale**: Keeping mint/melt transactions fully transparent ensures that token supply remains publicly auditable. Explorers and users can always verify the total circulating supply of any custom token by summing its mint and melt operations. Users who want amount privacy can move minted tokens into shielded outputs in a subsequent transaction.

## 4.4 Verification Pipeline

Verification is split into two phases, matching Hathor's existing architecture.

### Phase 1: Without Storage (Basic Verification)

Called during `verify_without_storage`. No UTXO lookups needed.

```
verify_shielded_outputs()
  |-- verify_commitments_valid()
  |     Checks: all commitments are 33-byte valid secp256k1 points
  |     Checks: asset_commitments (FullShielded) are valid points
  |     Checks: ephemeral_pubkeys are valid secp256k1 points
  |
  |-- verify_authority_restriction()              [Rule 7]
  |     Checks: no shielded output has authority bits set
  |
  |-- verify_range_proofs()                       [Rule 5]
  |     Checks: each shielded output's Bulletproof verifies against
  |             its commitment and generator (unblinded for Amount,
  |             blinded for Full)
  |
  |-- verify_trivial_commitment_protection()      [Rule 4, conservative]
  |     Checks: at least 2 shielded outputs (relaxed with storage)
  |
  |-- verify_shielded_fee()
        Checks: FeeHeader exists
        Checks: total_declared_fee >= shielded_fee (lower bound)
```

### Phase 2: With Storage (Full Verification)

Called during `verify` / `_verify_shielded_header`. Requires UTXO lookups to resolve input types.

```
_verify_shielded_header()
  |-- verify_surjection_proofs()                  [Rule 6]
  |     Builds surjection domain from input asset commitments:
  |       - Transparent inputs: derive_asset_tag(token_uid)
  |       - Shielded inputs: use on-chain asset_commitment
  |     Verifies each FullShieldedOutput's proof against the domain
  |
  |-- verify_shielded_balance()
  |     Collects all input commitments (shielded: direct, transparent: trivial)
  |     Collects all output commitments (shielded: direct, transparent: trivial)
  |     Appends fee entries as transparent outputs
  |     Verifies: sum(inputs) == sum(outputs)
  |
  |-- _verify_trivial_commitment_with_storage()   [Rule 4, relaxed]
        If any input is shielded: allow 1 shielded output
        Otherwise: require >= 2 shielded outputs

verify_token_rules(shielded_fee=X)                [Fee exact match]
  Existing fee verification, augmented with shielded_fee addend
  Checks: standard_fee + shielded_fee == fees_from_fee_header (exact)
```

### Token UID Normalization

The `_normalize_token_uid()` function handles the mismatch between Hathor's internal 1-byte HTR token UID (`b'\x00'`) and the crypto library's 32-byte requirement. HTR is padded with 31 zero bytes; custom tokens (already 32 bytes) pass through unchanged.

## 4.5 Fee Mechanism

### Fee Calculation

```python
shielded_fee = (n_amount_shielded * FEE_PER_AMOUNT_SHIELDED_OUTPUT)
             + (n_full_shielded   * FEE_PER_FULL_SHIELDED_OUTPUT)
```

Settings in `HathorSettings`:

| Setting | Default | Description |
|---------|---------|-------------|
| `FEE_PER_AMOUNT_SHIELDED_OUTPUT` | 1 | HTR base units per `AmountShieldedOutput` |
| `FEE_PER_FULL_SHIELDED_OUTPUT` | 2 | HTR base units per `FullShieldedOutput` |

These settings are gated by `ENABLE_SHIELDED_TRANSACTIONS`.

### FeeHeader Integration

Fees are declared in the existing `FeeHeader` mechanism. The `FeeHeader` entries are treated as transparent outputs in the homomorphic balance equation:

```
sum(C_in) == sum(C_out) + sum(C_fee_entry)
```

Each `C_fee_entry = fee_amount * H_token` for the corresponding token. This means the balance verification function does not need a separate `fee` parameter — fees are simply part of the output side.

### Two-Phase Fee Verification

1. **Without storage (lower bound)**: `total_declared_fee >= shielded_fee`. Cannot compute exact expected fee without storage (standard token fees depend on `chargeable_outputs` which requires UTXO lookups).
2. **With storage (exact match)**: `standard_fee + shielded_fee == fees_from_fee_header`. Both over-payment and under-payment are rejected.

### Shielded Fees Subsume Token Fees

Shielded outputs are not counted in `chargeable_outputs` for standard FEE-versioned token fee calculation. To prevent fee avoidance (shielding a token output to dodge `FEE_PER_OUTPUT`), the shielded fee rates are configured to be at least as large as the standard token output fee.

## 4.6 ECDH Recovery Mechanism

### Overview

Every shielded output contains an `ephemeral_pubkey` field (33 bytes). This enables the recipient to recover the committed value without any out-of-band communication.

### Sender Flow

1. Generate ephemeral keypair: `(e, E = e*G)` on secp256k1.
2. Obtain recipient's public key `P` (from prior transaction, payment request, or address book).
3. Compute shared secret: `s = SHA256(e * P)`.
4. Derive deterministic nonce: `nonce = SHA256("Hathor_CT_nonce_v1" || s)`.
5. Create range proof using `nonce` as the nonce key (not random).
6. For `FullShieldedOutput`: embed `token_uid(32B) || asset_blinding_factor(32B)` in the range proof message.
7. Store `E` (33 bytes, compressed) in the shielded output's `ephemeral_pubkey` field.

### Recipient Flow

1. Parse output script; check if address matches a wallet key.
2. Extract ephemeral pubkey `E` from the shielded output.
3. Compute shared secret: `s = SHA256(k * E)` (same result since `k*E = k*e*G = e*k*G = e*P`).
4. Derive nonce: `nonce = SHA256("Hathor_CT_nonce_v1" || s)`.
5. Call `rewind_range_proof(proof, commitment, nonce, generator)`:
   - `generator` = `derive_asset_tag(token_uid)` for `AmountShieldedOutput`
   - `generator` = `output.asset_commitment` for `FullShieldedOutput`
6. Returns: `(value, blinding_factor, message)`.
7. For `FullShieldedOutput`: extract `token_uid` and `asset_blinding_factor` from `message`, reconstruct expected asset commitment, and cross-check against on-chain value.

### Security Properties

- **Sighash binding**: The ephemeral pubkey is included in the transaction sighash, preventing MITM replacement.
- **Nonce uniqueness**: Each output uses a fresh ephemeral keypair.
- **Failed rewind**: Returns an error (not garbage) when the nonce is wrong — no false positives.
- **Forward secrecy**: Ephemeral keys are single-use.
- **Domain separation**: `"Hathor_CT_nonce_v1"` prefix isolates this derivation from other uses of the ECDH shared secret.
- **Token UID cross-check**: For `FullShieldedOutput`, the recovered `token_uid` is verified against the `asset_commitment` to prevent a malicious sender from embedding a fraudulent token UID.
- **No value logging**: Recovered amounts are never logged, even at DEBUG level.

## 4.7 Feature Activation

### Feature Flag

```python
# hathor/feature_activation/feature.py
class Feature(StrEnum):
    SHIELDED_TRANSACTIONS = 'SHIELDED_TRANSACTIONS'
```

### Settings

```python
# hathor/conf/settings.py
ENABLE_SHIELDED_TRANSACTIONS: FeatureSetting = FeatureSetting.DISABLED
```

`FeatureSetting` is an enum with values: `DISABLED`, `ENABLED`, `FEATURE_ACTIVATION`.

### Crypto Library Availability

At startup, if `ENABLE_SHIELDED_TRANSACTIONS != DISABLED`, the system validates that the native `hathor_ct_crypto` library is available via `validate_shielded_crypto_available()`. This prevents silent failures where all shielded output operations would fail at runtime.

Build command:
```bash
poetry run maturin develop --manifest-path hathor-ct-crypto/Cargo.toml --features python
```

## 4.8 Interaction with Other Features

### Silent Payments (Phase A)

The `ephemeral_pubkey` in shielded outputs serves double duty when Silent Payments are active:

- **Without SP**: The ephemeral key derives blinding factors only. The recipient address is visible in the script.
- **With SP**: The same ECDH shared secret derives both the one-time recipient address and the blinding factors. A single ECDH operation provides recipient privacy + blinding factor communication.

Combined: a shielded output hides the recipient (one-time address), the amount (Pedersen commitment), and optionally the token type (blinded asset tag).

### Ring Signatures / Nullifiers (Phase C)

Shielded outputs simplify decoy selection for ring signatures:

- **Without shielded outputs**: Decoys must match on amount (otherwise amount mismatch reveals the real input). This limits the anonymity set.
- **With shielded outputs**: All outputs are opaque commitments — any shielded output is a valid decoy regardless of hidden amount or token. Larger anonymity sets with simpler selection logic.

Surjection proofs naturally compose with ring signatures: the surjection domain includes all ring members (real + decoys), hiding which input is real.

### Nano Contracts

The `nc_caller` field in Nano Contract transactions identifies the calling address. Shielded outputs do not affect `nc_caller` — it remains a standard address. However, if Nano Contracts consume or produce shielded outputs in the future, the balance equation and proof generation would need to account for the contract's logic. This is out of scope for this RFC.

# Drawbacks
[drawbacks]: #drawbacks

1. **Transaction size increase.** A shielded output is ~770-930 bytes vs ~40 bytes for a transparent output (~20-25x larger). This increases storage, bandwidth, and propagation time. However, Hathor's logarithmic weight formula means the fee increase is moderate (not proportional to the size increase).

2. **Verification cost.** Bulletproof range proof verification takes ~1ms per proof. For a transaction with 4 shielded outputs, this adds ~4ms of CPU time per transaction. Batch verification and parallelization can amortize this, but it remains significantly more expensive than transparent output verification.

3. **Wallet complexity.** Wallets must implement ECDH key exchange, range proof rewind, blinding factor management, surjection proof generation, and fee computation. This is a substantial increase in wallet complexity compared to transparent-only transactions.

4. **Explorer capability reduction.** Block explorers lose the ability to display amounts and compute balances for shielded outputs. This fundamentally changes the user experience of public blockchain explorers, which are a key tool for transparency and debugging.

5. **Recipient public key requirement.** The sender must know the recipient's full compressed public key (not just the address hash) to establish the ECDH shared secret. This requires either a prior on-chain spend, a payment protocol, or a new address format.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why header-based (not a new TxVersion)?

A new `TxVersion` would require changes throughout the codebase — every `match vertex.version:` statement, serialization logic, feature gating, and verification path. The header-based approach reuses Hathor's existing header infrastructure (already used by `NanoHeader` and `FeeHeader`), attaching shielded outputs to standard transactions (`REGULAR_TRANSACTION` and `TOKEN_CREATION_TRANSACTION`) with zero structural changes. This means existing transaction processing, mempool logic, and DAG management continue to work unmodified.

## Why two output types (not one)?

`AmountShieldedOutput` (amount hidden, token visible) is substantially smaller and cheaper to verify than `FullShieldedOutput` (both hidden). Many use cases only need amount privacy — e.g., hiding salaries paid in HTR, where the fact that the token is HTR is not sensitive. Offering both tiers lets users pay only for the privacy they need.

## Why separate Bulletproofs (not aggregated)?

Aggregated Bulletproofs would require the prover to know all values and blinding factors for every output in the transaction. This is fundamentally incompatible with atomic swaps and multi-party transactions where each party independently constructs their outputs. The MPC workaround from Bünz et al. §4.3 is not implemented in `secp256k1-zkp` v0.11. Separate proofs also avoid permanently linking all outputs as created by the same party, and allow UTXO pruning. Performance is recovered through batch verification.

## Why secp256k1-zkp (not custom crypto)?

`secp256k1-zkp` is the industry-standard library for Confidential Transactions, maintained by Blockstream and used in production by Liquid/Elements. It provides battle-tested implementations of Pedersen commitments, Bulletproofs, and surjection proofs on the same secp256k1 curve Hathor already uses. Writing custom cryptographic primitives would be a security liability.

## Why ECDH rewind (not encrypted messages)?

Range proof rewinding is an established technique (used by Monero, Grin, Elements) that embeds recovery data inside the proof itself, adding zero bytes to the transaction. The alternative — encrypting amount/blinding data in a separate field — would increase transaction size and add a new encryption scheme to audit. Rewinding also provides a natural authentication mechanism: only the correct ECDH shared secret produces valid rewind results.

## Alternative: ZK-SNARKs (Zcash model)

Zcash's Sapling/Orchard circuits use Groth16/Halo 2 proofs to hide amounts, token types, and the sender simultaneously. While more powerful (combining Phases B and C), this approach requires:
- Trusted setup (Groth16) or substantially larger proofs (Halo 2 ~1.8KB vs Bulletproof ~675B)
- A different curve (BLS12-381 or Pallas/Vesta), incompatible with Hathor's secp256k1
- Complex circuit design and auditing
- Significantly longer proof generation time (~25-274ms vs ~2ms for Bulletproofs)

Hathor's phased approach achieves the same end-state with individually simpler, more auditable components.

## Alternative: Mimblewimble

Mimblewimble (used by Grin, Beam, Litecoin MWEB) provides amount hiding with transaction cut-through (pruning intermediate transactions). However:
- Mimblewimble requires interactive transaction construction (sender and receiver must communicate to build the transaction)
- It fundamentally changes the UTXO model and transaction structure
- It cannot support scripting or multi-asset transactions in their current form
- Hathor's DAG structure is incompatible with Mimblewimble's linear chain assumptions

## Impact of not doing this

Without shielded outputs, all Hathor transactions remain fully transparent. Users who need amount or token privacy would have no on-chain option, limiting Hathor's utility for business payments, DeFi, and personal finance. Competitors with privacy features (Monero, Zcash, Litecoin MWEB, Liquid) would have a structural advantage for privacy-sensitive use cases.

# Prior art
[prior-art]: #prior-art

## Liquid / Elements Confidential Assets

The primary inspiration for this design. Liquid (Blockstream's Bitcoin sidechain) implements Confidential Transactions with Pedersen commitments, Bulletproofs, and asset surjection proofs on secp256k1. Hathor's implementation uses the same `secp256k1-zkp` library and the same cryptographic constructions.

**Lessons learned:**
- Separate Bulletproofs per output are the practical choice (Liquid uses this).
- Asset surjection proofs are essential for multi-asset chains.
- ECDH-based range proof rewind works well for recovery.
- The two-output-type approach (amount-only vs. full) is a Hathor innovation not present in Liquid.

## Monero (RingCT + Bulletproofs)

Monero uses Pedersen commitments and Bulletproofs for amount hiding, combined with ring signatures for sender privacy. All transactions are mandatory CT.

**Differences from Hathor:**
- Monero uses Ed25519, not secp256k1.
- Monero's CT is mandatory; Hathor's is opt-in per output.
- Monero combines amount hiding with sender privacy (ring signatures) in a single protocol; Hathor separates these into independent phases.
- Monero does not support custom tokens.

**Lessons learned:**
- Bulletproof range proofs are production-proven at scale (Monero processes millions of CT transactions).
- ECDH-based recovery with deterministic nonces is the standard approach.
- Mandatory CT provides larger anonymity sets but increases chain size.

## Zcash (Sapling / Orchard)

Zcash uses ZK-SNARKs (Groth16 in Sapling, Halo 2 in Orchard) to provide full transaction privacy: hidden amounts, hidden token types, and hidden senders, all in a single proof.

**Differences from Hathor:**
- Zcash uses different curves (BLS12-381, Pallas/Vesta).
- Zcash's proofs are more expensive to generate (~25ms Groth16, ~274ms Halo 2 vs ~2ms Bulletproof).
- Zcash requires circuit compilation and trusted setup (Groth16) or larger proofs (Halo 2).
- Zcash's shielded pool is separate from its transparent pool (different address types); Hathor mixes them freely.

**Lessons learned:**
- Opt-in privacy (Zcash's shielded pools) means most transactions remain transparent, reducing the effective anonymity set. Hathor should encourage shielded usage to grow the anonymity set.
- ZIP 317 fee structure (per-action fees) is a good model for incentive alignment.

## Grin / Beam (Mimblewimble)

Mimblewimble protocols use Pedersen commitments with transaction cut-through for amount hiding and chain compaction.

**Not adopted because:** Interactive transaction construction, incompatibility with scripting and multi-asset support, and incompatibility with Hathor's DAG structure.

## Litecoin MWEB

Litecoin's MimbleWimble Extension Blocks (activated May 2022) add opt-in confidential transactions via a sidechain-like extension block.

**Relevant parallels:**
- Opt-in privacy (like Hathor).
- Shield/unshield operations at the boundary between transparent and MWEB pools.
- ECDH-based stealth addresses for recipient privacy.

**Differences:** MWEB is a separate block structure with different consensus rules; Hathor integrates shielded outputs directly into the existing transaction format via headers.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Shielded address format.** Option A (compact, 53-byte payload) vs Option B (full keys, 66-byte payload). The key tradeoff is address length vs. Silent Payments compatibility. This should be resolved before wallet implementations begin.

2. **Final fee amounts.** The placeholder values (`FEE_PER_AMOUNT_SHIELDED_OUTPUT = 1`, `FEE_PER_FULL_SHIELDED_OUTPUT = 2`) are not final. Production values should be determined based on actual verification cost differentials, network economics, and desired incentive structure.

3. **Batch verification optimization timeline.** The implementation currently verifies range proofs individually. Batch verification (`verify_multi`) would reduce CPU cost by an estimated 30-50% for multi-output transactions. The secp256k1-zkp API supports this, but the integration timeline is not yet determined.

4. **Light client scanning support.** Full nodes must verify all proofs, but light clients need an efficient protocol for detecting their own shielded payments without downloading every transaction. Possible approaches include SP-style scanning filters, trusted server-side scanning, or compact proof summaries.

5. **Shielded Nano Contract interactions.** How should Nano Contracts interact with shielded outputs? Can a contract consume shielded inputs or produce shielded outputs? What are the implications for contract state visibility? This is deferred to a future RFC.

6. **Fee adjustment mechanism.** Should per-output fees be fixed consensus constants or adjustable via feature activation? Fixed constants are simpler but cannot adapt to changing verification costs or network conditions.

# Future possibilities
[future-possibilities]: #future-possibilities

## Sender Privacy (Phase C)

Ring signatures or nullifier-based protocols would hide which input is being spent, completing the privacy trifecta (hidden recipient + hidden amount/token + hidden sender). Shielded outputs simplify Phase C by making all outputs valid decoys regardless of their hidden amount or token.

## Aggregated Bulletproofs for Multi-Party

If a multi-party computation (MPC) protocol for aggregated Bulletproofs becomes available in `secp256k1-zkp`, transactions where all outputs are controlled by the same party (the common case for non-atomic-swap transactions) could use a single aggregated proof, reducing total proof size logarithmically.

## Shielded Fee Amounts

The fee amount is currently transparent. A future extension could allow the fee itself to be committed — the sender would prove (via range proof) that the committed fee exceeds the minimum, without revealing the exact amount. This would hide the number of shielded outputs (which is currently inferrable from the fee).

## Cross-Chain Atomic Swaps with CT

Shielded outputs on both sides of a cross-chain atomic swap would hide the amounts exchanged. Since each party independently balances their own blinding factors (see Section 4.1), no blinding factor exchange is needed. The atomic swap protocol only needs to coordinate the hash-time-lock, not the privacy layer.

## View Key Delegation for Compliance

Users could derive a "view key" that allows a designated party (auditor, regulator, exchange) to decrypt all shielded outputs addressed to them, without granting spending authority. This is analogous to Monero's view keys and could be implemented by sharing the scan private key.

## Tiered Privacy Fees

As the privacy stack matures, fees could be tiered by privacy level: transparent < amount-shielded < full-shielded < ring-shielded. This naturally extends the per-output-type fee model established in this RFC.
