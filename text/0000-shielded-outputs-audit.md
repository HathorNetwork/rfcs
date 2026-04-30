- Feature Name: shielded_outputs_audit
- Start Date: 2026-04-29
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Hathor Labs

# Summary
[summary]: #summary

This RFC extends [Shielded Outputs](0000-shielded-outputs.md) with a per-transaction **binding signature** that enables network-wide supply auditability. The existing `ShieldedOutputsHeader` is augmented with a 33-byte kernel-excess point `E_tx = e_tx · G` together with a 64-byte Schnorr signature proving knowledge of the discrete log. The signature simultaneously serves as a non-interactive proof of value-balance and binds the proof to the transaction. A chain-wide audit equation then permits any external observer to verify, in a single curve-equation per token, that no inflation has occurred across the entire transaction history. The construction is structurally identical to Zcash Sapling's binding signature, generalized to Hathor's per-token confidential transactions. As a side benefit, the change removes the "wallet forces `e_tx = 0`" pattern from the current shielded design, allowing every output's blinding factor to be independently uniformly random — unlocking receiver-chosen blindings, multi-party transaction construction, and composable output addition.

# Motivation
[motivation]: #motivation

The shielded outputs feature hides individual UTXO amounts behind Pedersen commitments. While per-transaction balance is enforced by every full node during verification, **chain-wide supply auditability is lost**: an external monitor can no longer sum cleartext UTXO amounts and compare against expected supply.

This matters for two distinct populations:

1. **Inflation-bug monitors.** A bug in any verifier path — range proof check, surjection proof check, balance equation, or any future addition — could allow a transaction to create value from nothing. Without a chain-wide invariant, such a bug remains silent until it is exploited and economic damage manifests. With a chain-wide audit equation, a single check against the UTXO set detects any past inflation regardless of which verifier path was buggy.

2. **External auditors and explorers.** Token issuers, regulators, exchanges, and explorers need to verify total circulating supply per token. For tokens whose issuer chooses transparent mint/melt, supply is currently auditable from cleartext fields alone. But this depends on the issuer's good faith. A protocol-level audit equation makes supply verification independent of issuer choices for the entire shielded-pool history.

The current shielded design supports per-tx balance via the equation `Σ C_in = Σ C_out + fee · H_HTR`, with the wallet constructing the last shielded output's blinding factor as the residual `r_n = Σ r_in − Σ r_others` so that the per-tx kernel excess is structurally zero. This per-tx zero-excess pattern has three consequences:

- **No chain-wide telescoping.** The kernel excess is never published, so the standard CT/Mimblewimble chain-wide audit construction does not apply.
- **Within-tx blinding correlation.** If any `n−1` blindings of a transaction leak (RNG flaw, partial seed compromise, side channel), the last output's blinding is fully determined.
- **Sender must know all blindings.** The construction requires the sender to compute the residual blinding, which precludes Sapling-style receiver-derived blindings, multi-party transaction construction, and easy output addition.

This RFC introduces a per-tx binding signature that simultaneously closes the audit gap and removes the construction constraints, at a cost of 98 bytes per shielded transaction.

**Expected outcome.** Any external observer, given the entire chain history and the current UTXO set, can verify in a single curve-point equation per token that no value has been created or destroyed across the entire shielded-pool history. Wallet construction simplifies: every output's blinding factor is independently uniformly random. The change is a backward-incompatible extension to `ShieldedOutputsHeader` (gated by network upgrade) and removes the now-redundant `UnshieldBalanceHeader`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## 1. The Audit Equation

After this RFC, an external monitor (anyone holding the chain history and the current UTXO set) can verify total supply per token via:

```
Σ C_utxo  ==  total_supply · H_token  +  Σ_tx E_tx
```

- `Σ C_utxo` — sum of all unspent value commitments. Transparent UTXOs are treated as commitments with `r = 0`, i.e., `C = v · H_token`. Shielded UTXOs use their on-chain commitment directly.
- `total_supply` — the publicly computable expected supply, derived from cleartext fields: `initial + Σ mint − Σ melt ± Σ shield/unshield deltas`.
- `Σ_tx E_tx` — the running sum of every shielded transaction's binding-signature public key (`E_tx = e_tx · G`). Maintained incrementally as a single 33-byte curve point.

Both sides are points on secp256k1. The verifier computes both and checks equality. If any historical transaction inflated the supply (for any reason — verifier bug, broken range proof, miscomputed mint), the equation fails.

The equation is the multi-asset generalization of Zcash Sapling's chain-wide value balance. The same machinery proves both per-tx balance (during verification) and chain-wide balance (during audit) — see [Section 4.4](#44-the-binding-signature-as-proof-of-balance).

## 2. The Binding Signature

Each shielded transaction's `ShieldedOutputsHeader` is extended with three trailing fields:

| Field | Size | Purpose |
|---|---|---|
| `excess_point` | 33 B | Kernel excess `E_tx = e_tx · G` — public key commitment to the transaction's blinding-factor balance |
| `signature_R` | 33 B | Schnorr signature commitment `R = k · G` |
| `signature_z` | 32 B | Schnorr signature response `z = k + c · e_tx (mod q)` |

Together these prove knowledge of `e_tx` such that `excess_point = e_tx · G`, bound to the transaction sighash. The verifier checks two things:

```
1. Σ C_out − Σ C_in − fee · H_HTR  ==  E_tx       (balance equation)
2. Schnorr.Verify(E_tx, sighash, signature)        (binding signature)
```

The first check enforces homomorphic balance against the published `E_tx`. The second check proves that `E_tx` lies on the `G`-subgroup with a known discrete log — which forces value terms in the LHS to cancel exactly, since `G` and `H_token` are independent generators with no known DL relation.

## 3. What Changes for Wallets

The wallet no longer needs to compute the last output's blinding factor as a residual. Instead:

```
Old construction:
  for i in 0..n-1:    r_i  ←$  Z_q
  r_n  :=  Σ r_in  −  Σ_{i<n} r_i           # forced residual

New construction:
  for i in 0..n:      r_i  ←$  Z_q           # all independent
  e_tx  :=  Σ r_out  −  Σ r_in
  E_tx  :=  e_tx · G
  σ      :=  Schnorr.Sign(e_tx, sighash)
```

Every output's blinding is independently uniformly random. The per-tx excess `e_tx` is computed after blinding selection, and the binding signature attests to it.

This unlocks four useful patterns immediately:

1. **Receiver-chosen blindings.** A receiver supplies their own commitment with `r` derived from their viewing key. The sender aggregates `r_recipient · G` (a public point) without ever seeing `r_recipient`. Enables deterministic wallet recovery, payment-proof schemes, and stealth-address improvements.

2. **Multi-party construction.** Sender and receiver construct their pieces separately. The protocol sums their contributions to `e_tx` and produces a single binding signature.

3. **Composable output addition.** Adding a fee-bump output, dummy mixing output, or memo output no longer triggers recomputing the last output's blinding (and re-running its range proof).

4. **No within-tx blinding correlation.** Leaking `n−1` blindings reveals the last under the old construction; under the new, each blinding is independent.

## 4. What Changes for Verifiers

Verifiers gain one additional check per shielded transaction (the binding signature) and one additional invariant per shielded transaction (the balance-equation residual must equal `E_tx` exactly). The existing per-tx balance check `Σ C_in = Σ C_out + fee · H_HTR` is replaced by `Σ C_in = Σ C_out + fee · H_HTR + E_tx`.

External monitors gain a chain-wide audit capability. A reference implementation requires:

- Maintenance of a running `Σ_tx E_tx` curve point (33 B), updated as new shielded transactions are processed.
- Maintenance of a running `total_supply` per token, updated from cleartext mint/melt/shield/unshield events.
- On demand: sum `Σ C_utxo` from the current UTXO set, compute the audit equation, check equality.

The audit can be run by anyone with chain access; full-node trust is not required for the audit step.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 4.1 Extended `ShieldedOutputsHeader`

The existing `ShieldedOutputsHeader` (`0x12`) is augmented with three trailing fields carrying the kernel excess and binding signature. The fields are mandatory whenever the header is present (i.e., for every transaction with shielded outputs); transactions without shielded outputs never carry the header and never carry a binding signature.

**Updated layout** (additions marked ★):

| Field | Size | Description |
|---|---|---|
| Header ID | 1 B | `0x12` (`VertexHeaderId.SHIELDED_OUTPUTS_HEADER`) |
| `num_outputs` | 1 B | Number of shielded outputs (max 32) |
| Outputs | variable | Concatenated serialized outputs |
| `excess_point` | 33 B | Compressed secp256k1 point `E_tx = e_tx · G` ★ |
| `signature_R` | 33 B | Compressed secp256k1 point `R = k · G` ★ |
| `signature_z` | 32 B | Scalar `z = k + c · e_tx  (mod q)` ★ |

**Added size**: 98 B fixed per shielded transaction.

The signature is encoded in `(R, z)` form (33 + 32 = 65 B) rather than `(c, z)` form to permit batch verification across multiple signatures (compute `z·G − R == c·E_tx` as a multi-exp).

The existing [`UnshieldBalanceHeader`](../../hathor-core-4/hathorlib/hathorlib/headers/unshield_balance_header.py) (`0x13`) is **removed**. Its `excess_blinding_factor` field carried the same per-tx information, but as a raw scalar — exposing it directly. After this RFC lands, full-unshield transactions use the extended `ShieldedOutputsHeader` like any other shielded transaction. The header ID `0x13` is retired and not reused.

### Wire Format

```
ShieldedOutputsHeader serialization:
  header_id(1B=0x12) | num_outputs(1B) | outputs(var)
                    | excess_point(33B) | signature_R(33B) | signature_z(32B)
```

All multi-byte fields are big-endian. Curve points are SEC1 compressed encoding. The 98 B trailer is fixed-size and immediately follows the variable-size outputs blob.

### Sighash Coverage

The transaction sighash includes everything that uniquely determines the transaction body:

- All transparent inputs (outpoint references and scripts)
- All transparent outputs (`amount`, `token_data`, `script`)
- All shielded outputs (`mode`, `commitment`, `script`, `token_data` or `asset_commitment`, `ephemeral_pubkey`)
- All other headers' canonical fields: `FeeHeader`, `MintHeader`, `MeltHeader`, etc.
- The `excess_point` field of `ShieldedOutputsHeader`

The sighash **excludes**:

- `signature_R` and `signature_z` from `ShieldedOutputsHeader` (cannot include the signature in its own message — standard practice, analogous to Bitcoin excluding `scriptSig` from sighash)
- Range proofs and surjection proofs (verified independently; do not affect the binding)

In implementation terms: the sighash is computed over the `ShieldedOutputsHeader` byte range from `header_id` through the end of `excess_point`, excluding the trailing 65 B `(signature_R, signature_z)`. The sighash construction matches the existing rules in [Shielded Outputs Section 4.2](0000-shielded-outputs.md#42-data-structures), augmented with `excess_point`.

## 4.2 Cryptographic Primitives

### Schnorr Signature Scheme

The binding signature uses Schnorr over secp256k1, with deterministic nonces and Fiat-Shamir challenges via SHA-256.

**Sign** (given private scalar `s = e_tx`, message `m = sighash`):

```
1. k     :=  HMAC-SHA256(key=s, msg="HathorBindingSig/v1" || m)   # RFC 6979-style
2. R     :=  k · G
3. c     :=  SHA-256("HathorBindingSig/v1/c" || R || P || m)  mod q
4. z     :=  k + c · s  (mod q)
5. σ     :=  (R, z)
```

**Verify** (given public point `P = E_tx`, message `m = sighash`, signature `σ = (R, z)`):

```
1. Reject if R or P fail point validation (not on curve, or identity)
2. Reject if z is not in [0, q)
3. c     :=  SHA-256("HathorBindingSig/v1/c" || R || P || m)  mod q
4. Accept iff  z · G  ==  R + c · P
```

The deterministic nonce derivation prevents the catastrophic nonce-reuse attack that would otherwise leak `s` (and therefore `Σ r_out − Σ r_in`) on any RNG hiccup. The domain separator `"HathorBindingSig/v1"` prevents cross-protocol reuse of signatures.

### Curve Choice

secp256k1 throughout, matching the existing shielded outputs design. The base generator `G` and per-token NUMS generators `H_token` retain the discrete-log independence assumption from the parent RFC.

### Batch Verification

Multiple binding signatures across a block can be batch-verified. Given signatures `(R_i, z_i)` over messages `m_i` for public keys `P_i`:

```
Pick random scalars α_1, …, α_n  ←$  Z_q
Compute c_i = SHA-256(...||R_i||P_i||m_i) mod q
Check:  Σ α_i · z_i · G  ==  Σ α_i · (R_i + c_i · P_i)
```

A single multi-exponentiation replaces `n` separate verifications. The `α_i` randomization prevents an adversary from constructing forgeries that cancel in the sum.

## 4.3 Construction: Wallet Algorithm

The wallet algorithm for building a shielded transaction is updated as follows. Differences from [Shielded Outputs Section 5](0000-shielded-outputs.md#guide-level-explanation) are marked.

```
1. Select transparent and shielded inputs.
2. For each shielded input, retrieve the wallet-side blinding factor r_i.
3. Compute Σ r_in_shielded.
4. For each output (shielded or transparent change):
     - Choose value v_i.
     - For shielded: choose r_i ←$ Z_q INDEPENDENTLY.   ★ CHANGED — no residual
     - Compute commitment C_i = v_i · H_token + r_i · G.
     - Generate range proof.
     - For FullShieldedOutput, choose asset blinding r_asset_i and generate
       surjection proof.
5. Compute e_tx := Σ r_out_shielded − Σ r_in_shielded.   ★ NEW
6. Compute E_tx := e_tx · G.   ★ NEW
7. Build the transaction body, including ShieldedOutputsHeader with outputs
   and excess_point = E_tx (signature_R, signature_z left as placeholder zeros).
8. Compute sighash including E_tx, excluding signature_R/signature_z
   (see Section 4.1 sighash coverage).
9. Sign sighash with e_tx via Schnorr → produces (R, z).   ★ NEW
10. Patch ShieldedOutputsHeader's signature_R, signature_z fields with (R, z).   ★ NEW
11. Sign each transparent input separately as before.
12. Submit transaction.
```

Step 4 no longer special-cases the last output. Step 5–10 add the binding signature.

For `FullShieldedOutput` with asset blinding, an analogous binding signature would be needed if cross-asset privacy is to be preserved without a separate "asset balance" check. This RFC focuses on the value-balance binding signature; asset-balance auditability is discussed in [Future Possibilities](#future-possibilities).

## 4.4 The Binding Signature as Proof of Balance

This subsection contains the cryptographic argument. It is not optional reading: the signature does cryptographic work that the balance equation alone does not.

### Why the Equation Alone Is Insufficient

Suppose we publish `E_tx` without a signature. The verifier checks:

```
Σ C_out − Σ C_in − fee · H_HTR  ==  E_tx
```

The verifier can simply compute the LHS from the published commitments and accept whatever curve point falls out as `E_tx`. The check is trivially satisfied for any transaction, including inflating ones.

Concretely, an attacker constructs an inflated output `C_inflate = Δv · H + r' · G` with extra value `Δv` not matched by any input. The residual becomes:

```
Σ C_out − Σ C_in − fee · H_HTR  =  Δv · H  +  e_tx · G
```

This is a curve point with both an `H`-component and a `G`-component. Without a constraint on `E_tx`'s structure, the attacker simply declares `E_tx := Δv · H + e_tx · G` and the equation passes.

The chain-wide audit does not catch this either: the `Δv · H` term propagates symmetrically into `Σ_tx E_tx` and into `Σ C_utxo − total_supply · H_token`, so the equation rebalances trivially. Audit is silent.

### Why the Schnorr Signature Closes the Gap

The Schnorr signature on `E_tx` proves the prover knows a scalar `s` such that `E_tx = s · G`. Combined with the balance equation:

```
Σ C_out − Σ C_in − fee · H_HTR  =  s · G       (s known to prover)
```

The right-hand side lies purely on the cyclic subgroup generated by `G` — it has **no `H_token`-component** for any token. Since `G` and `H_token` are NUMS generators with no known discrete-log relation, the only way for the LHS to equal a point with no `H`-component is for the value terms to cancel exactly:

```
Σ v_out_token  +  fee · 𝟙[token = HTR]  =  Σ v_in_token         for every token
```

An inflating prover would need to compute the discrete log of `Δv · H_token` with respect to `G` to construct a valid signature — the hardness assumption underpinning the entire scheme.

The signature is therefore a **non-interactive proof of value-balance**, not authorization. Authorization for spending shielded UTXOs is handled separately by the existing per-input mechanisms in the parent RFC.

### Telescoping for Chain-Wide Audit

Every blinding factor `r_o` of output `o` appears in `Σ_tx e_tx` exactly:

| State of `o` | Net contribution to `Σ_tx e_tx` |
|---|---|
| Spent | `+r_o − r_o = 0` (created as output, consumed as input) |
| Unspent (UTXO) | `+r_o` |

Therefore `Σ_tx e_tx = Σ r_utxo`, and:

```
Σ_tx E_tx  =  (Σ r_utxo) · G
           =  Σ C_utxo  −  total_supply · H_token
```

The verifier maintains a single 33 B running point `Σ_tx E_tx` and a per-token cleartext supply tally. Audit is a single curve-equation check per token.

## 4.5 Updated Transaction Rules

This RFC modifies several rules from [Shielded Outputs Section 4.3](0000-shielded-outputs.md#43-transaction-rules).

### Rule 3 (REPLACED): Binding Signature Required

Every `ShieldedOutputsHeader` MUST carry the trailing `excess_point`, `signature_R`, and `signature_z` fields. The `excess_point` MUST equal `Σ C_out − Σ C_in − fee · H_HTR` evaluated over all transparent and shielded inputs and outputs (transparent contributing trivial commitments). The signature `(R, z)` MUST verify against `excess_point` and the transaction sighash under the Schnorr scheme defined in [Section 4.2](#42-cryptographic-primitives).

Transactions without shielded outputs do not include a `ShieldedOutputsHeader` and therefore carry no binding signature.

The previous "blinding factors must balance" rule is removed: blindings no longer need to balance to zero, and the wallet does not force the last output's blinding as a residual.

### Rule 4 (RELAXED): Trivial Commitment Protection

The previous rule required at least 2 shielded outputs when all inputs are transparent, to prevent the wallet's residual blinding factor from being trivially zero. With independent uniformly-random blindings, the trivial-commitment cryptographic concern is gone — the commitment is properly hiding.

However, the **structural amount leak** remains: when all inputs are transparent and there is exactly one shielded output, the value `v_o` is publicly computable from the balance equation:

```
v_o  =  Σ v_in_transparent  −  Σ v_out_transparent  −  fee
```

This is a property of the balance constraint, not the wallet construction, and cannot be removed without breaking balance. The amount privacy concern remains, but it is now a *user* concern (the user constructed a transaction whose value is forced public by balance), not a *protocol* concern (the commitment itself is properly hiding).

**Updated rule**: at least 2 shielded outputs are RECOMMENDED but not REQUIRED when all inputs are transparent. Wallets SHOULD warn users that single-shielded-output shield-in transactions reveal the shielded output's value via balance. Consensus MAY choose to enforce the recommendation; this RFC takes no position. (See [Unresolved Questions](#unresolved-questions).)

### Rule 7 and Rule 8: Unchanged

Authority outputs remain transparent. Mint/melt transactions cannot have shielded outputs. These constraints are independent of the binding signature.

## 4.6 Verification Pipeline

The verification pipeline from the parent RFC is augmented with binding-signature checks.

### Phase 1: Without Storage

```
verify_shielded_outputs()
  |-- verify_commitments_valid()
  |-- verify_authority_restriction()              [Rule 7]
  |-- verify_range_proofs()                       [Rule 5]
  |-- verify_binding_signature_fields_well_formed()  [NEW — Rule 3]
  |     Checks: excess_point is a valid secp256k1 point
  |     Checks: signature_R is a valid secp256k1 point
  |     Checks: signature_z is in [0, q)
  |
  |-- verify_shielded_fee()
        Checks: FeeHeader exists, total fee ≥ shielded_fee
```

### Phase 2: With Storage

```
_verify_shielded_header()
  |-- verify_surjection_proofs()                  [Rule 6]
  |
  |-- verify_balance_against_excess()             [NEW — replaces verify_shielded_balance]
  |     Builds LHS = Σ C_out − Σ C_in − fee · H_HTR (transparent contribute trivial commitments)
  |     Checks: LHS == ShieldedOutputsHeader.excess_point
  |
  |-- verify_binding_signature()                  [NEW — Rule 3]
  |     Computes sighash (excluding signature_R, signature_z)
  |     Verifies Schnorr signature against (excess_point, sighash, signature_R, signature_z)

verify_token_rules(shielded_fee=X)                [Unchanged]
```

The previous `verify_shielded_balance()` is replaced by `verify_balance_against_excess()`. The signature check is new.

### Audit Verifier Algorithm

The audit is run by external monitors, not by full nodes during transaction processing. It maintains state across blocks:

```
State:
  excess_sum: secp256k1.Point          # Σ_tx E_tx, init = identity
  total_supply: dict[token_uid, int]    # init from genesis allocations
  utxo_commitments: dict[outpoint, Point]  # current UTXO set, value commitments

Update on each new transaction tx:
  if tx has ShieldedOutputsHeader:
    excess_sum += tx.shielded_outputs_header.excess_point
  for output o in tx.outputs:
    utxo_commitments[outpoint(o)] = commitment_of(o)
  for input i in tx.inputs:
    del utxo_commitments[outpoint(i)]
  if tx contains MintHeader entry for token T with amount A:
    total_supply[T] += A
  if tx contains MeltHeader entry for token T with amount A:
    total_supply[T] -= A

Audit at any block height:
  Σ_C = sum(utxo_commitments.values())
  expected = Σ_token (total_supply[token] · H_token) + excess_sum
  return Σ_C == expected
```

Maintenance cost: O(1) per transaction (one point addition for the excess sum, one map update per UTXO change). Storage cost: O(|UTXO set|) — same as a full node's UTXO state, plus a single 33 B running point.

## 4.7 Migration

Two paths, depending on whether shielded outputs have already been deployed at the time of this RFC's activation.

**Path A (preferred): activate together.** This RFC and the parent shielded outputs RFC activate at the same network upgrade height. From genesis-of-shielded-outputs onward, every `ShieldedOutputsHeader` includes the binding-signature trailer and `UnshieldBalanceHeader` is never deployed. No legacy shielded transactions exist; no migration logic needed.

**Path B (if shielded outputs precede this RFC): hard-fork transition.** A network upgrade height `H_audit` is announced. Before `H_audit`, shielded transactions follow the parent RFC's "wallet forces `e_tx = 0`" pattern: `ShieldedOutputsHeader` has no trailer, and full-unshield transactions use `UnshieldBalanceHeader` (`0x13`). After `H_audit`, the extended `ShieldedOutputsHeader` layout is mandatory and `UnshieldBalanceHeader` is rejected. Pre-`H_audit` shielded outputs in the UTXO set contribute to `Σ C_utxo` in the audit equation, but the chain-wide telescoping covers only post-`H_audit` transactions. The audit equation becomes:

```
Σ C_utxo  ==  total_supply · H_token  +  Σ_{tx >= H_audit} E_tx  +  legacy_offset
```

where `legacy_offset = Σ r_pre_H_audit_utxos · G`, fixed at `H_audit` and computed once by summing the pre-fork UTXO commitments and subtracting `pre_fork_supply · H_token`. This single "snapshot offset" point is published as part of the network upgrade and lets the chain-wide equation remain valid across the transition.

Wallets and full nodes MUST be updated before `H_audit`. Old wallets producing pre-RFC `ShieldedOutputsHeader` (without trailer) after `H_audit` will produce invalid transactions, rejected by full nodes for missing binding-signature fields. The deserializer distinguishes the two formats by network-upgrade context — there is no in-band version byte, since the header layout is determined by the activation height.

# Drawbacks
[drawbacks]: #drawbacks

- **98 byte per-tx wire cost.** Every shielded transaction grows by a fixed-size trailer in `ShieldedOutputsHeader`. For a transaction with two shielded outputs (~1.6 KB), this is ~6% overhead. Negligible per-tx, but accumulates over chain history.

- **One additional cryptographic verification per transaction.** Schnorr verification is a single multi-exp on secp256k1 — fast, but not free. Batch verification across a block amortizes the cost.

- **Increased complexity for wallet implementations.** Wallets must compute `e_tx`, derive the binding-signature key, sign with deterministic nonces, and serialize the extended `ShieldedOutputsHeader` trailer. The change is well-isolated (mechanical sign/verify, fixed-size fields) but is still an additional surface area.

- **Single-output shield-in privacy footgun.** The structural amount leak when all inputs are transparent and there is one shielded output remains. This RFC documents the issue and recommends a wallet warning, but consensus does not enforce a multiple-shielded-output rule. A user who constructs such a transaction reveals the shielded value through the balance equation, regardless of the binding signature.

- **No per-token isolation.** The chain-wide audit equation is a single check across all tokens jointly. A monitor cannot isolate HTR auditability from custom token correctness without either (a) constraining HTR to transparent operations or (b) adding per-asset binding signatures (loses asset privacy). This is discussed in [Future Possibilities](#future-possibilities).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why publish `E_tx` as a curve point rather than the scalar `e_tx`?

A scalar `e_tx` would also prevent inflation: the verifier checks `Σ C_out − Σ C_in − fee · H_HTR = e_tx · G`, computing `e_tx · G` from the published scalar; the RHS is structurally on the `G`-subgroup with known discrete log. No Schnorr signature would be needed for inflation prevention. Wire format: 32 B vs 33 + 64 = 97 B.

The privacy cost of publishing the scalar is unacceptable:

1. **Direct blinding-factor leak in the single-output corner case.** Where the point form leaks only `r_o · G` (recovering `r_o` requires solving DLP), the scalar form leaks `r_o` directly. Defense-in-depth lost.

2. **`Σ r_utxo` becomes a publicly computable scalar.** Any single UTXO blinding leak via side channel composes with the chain-wide sum to reveal further blindings. Point form forces an attacker to solve DLP for each composition.

3. **Receiver-chosen blinding is impossible.** Sapling-style receivers contribute `r_recipient · G` (a public point from their viewing key) without revealing `r_recipient`. Sender aggregation works on points, not on scalars.

4. **No path to multi-party / non-interactive aggregation.** Combining contributions from multiple parties requires summing kernel excesses without revealing individual contributions. Point form supports this; scalar form does not.

5. **Cross-tx statistical analysis** of published scalars could expose RNG biases. Points are uniform group elements with no exploitable structure.

The 65 B saved are not worth losing defense-in-depth, receiver-chosen blindings, and aggregation friendliness.

## Why a Schnorr signature rather than a different proof of DL knowledge?

Schnorr is the standard proof of knowledge of discrete log, with three desirable properties: 64 B size, single multi-exp verification, batch-verifiable. Alternatives considered:

- **ECDSA**: also 64 B, also batch-verifiable in some constructions, but more complex and lacks linearity (cannot trivially aggregate). Schnorr is preferable.
- **EdDSA / Ed25519**: requires a different curve. The shielded outputs design uses secp256k1 throughout; introducing a second curve is unwarranted complexity.
- **BLS signature**: smaller (48 B) and aggregatable, but requires pairing-friendly curves (BLS12-381) and pairings for verification — expensive and inconsistent with the rest of the shielded design.
- **Bulletproofs proof of knowledge**: overkill (and 600+ B).

Schnorr over secp256k1 with deterministic nonces, `(R, z)` form for batch-friendly verification.

## Why fold the binding signature into `ShieldedOutputsHeader` rather than introducing a new header?

The binding signature is required iff `ShieldedOutputsHeader` is present (Rule 3). Two headers that always co-occur are arguably one header. Folding has three concrete benefits and one cost.

Benefits:

1. **Saves the 1 B header tag** — small but free, and accumulates over chain history.
2. **One less header type** in `VertexHeaderId`. Hathor's enum stays smaller; one fewer mental object for wallet authors and verifier readers.
3. **Replaces the deprecated `UnshieldBalanceHeader` cleanly.** That header was a separate balance-data header; extending the precedent with a third one (`BalanceSignatureHeader`) is the wrong direction. Folding lets us collapse balance attestation into the existing shielded-outputs envelope and retire `UnshieldBalanceHeader` outright.

Cost:

- **Mild conceptual misnaming.** The binding signature is a transaction-level value-balance proof, not specifically about shielded outputs — but in practice the two are 1-to-1 in this RFC, and the misnaming is paid for by simplicity. If a future RFC adds asset-balance binding signatures or block-level aggregated signatures, those can be addressed when the time comes (either by extending `ShieldedOutputsHeader` further or by introducing a separate header for aggregation).

A separate `BalanceSignatureHeader` was considered earlier in this RFC's drafting and rejected on the grounds above.

## Why not enforce single-shielded-output protection?

The previous Rule 4 required at least 2 shielded outputs when all inputs are transparent, originally to avoid the cryptographic trivial-commitment issue caused by `r_n = 0`. With independent random blindings under this RFC, the cryptographic issue is gone; the remaining issue is a structural amount-balance leak that no wallet rule can fully prevent.

Two arguments for *not* enforcing:

- **The leak is structural.** `v_o = Σ v_transparent_in − Σ v_transparent_out − fee` is a balance identity. Adding a second shielded output hides individual values but not their sum, and a wallet that constructs the second output's value as a fixed multiple of the first is no different from a single-output transaction in terms of leaked information.
- **User intent.** A user who constructs a single-shielded-output shield-in may have deliberate reasons (testing, atomic commitments, low-privacy regimes) and should not be blocked by consensus.

Two arguments for enforcing:

- **Footgun reduction.** Most users will not understand the balance leak and will assume "shielded = private."
- **Cheap to enforce.** Adding a second 700 B output is small overhead.

This RFC takes a permissive stance (warn but don't enforce) and flags the question as unresolved. See [Unresolved Questions](#unresolved-questions).

## Impact of not doing this

Without a chain-wide audit equation:

- Inflation bugs in any verifier path remain silent until economically exploited.
- External auditors must trust the per-tx verification of every full node.
- Receiver-chosen blindings, multi-party transaction construction, and composable output addition are blocked at the wallet level.
- The `UnshieldBalanceHeader` scalar-excess publication remains the only chain-public excess data — covering only full-unshield transactions and exposing the scalar directly.

Cumulatively: weaker safety story for shielded outputs, less wallet flexibility, and a privacy footgun (scalar excess in `UnshieldBalanceHeader`) that is preferable to remove anyway.

# Prior art
[prior-art]: #prior-art

## Zcash Sapling (NU2, 2018)

Sapling's **binding signature** is the direct precedent. Sapling publishes a per-tx `cv_balance` (value commitment to the net balance) and a Schnorr signature over the sighash with `cv_balance · G` as the public key. The construction enforces value-balance non-interactively and is structurally identical to this RFC's binding signature.

Differences:

- Sapling has a single shielded pool with one value generator (homogeneous shielded value); Hathor has per-token generators (multi-asset CT).
- Sapling uses Jubjub curve (an Edwards curve embedded inside BLS12-381 for SNARK efficiency); Hathor uses secp256k1.
- Sapling combines binding signatures with zk-SNARK-based spend authorization; Hathor uses range proofs and surjection proofs separately.

The audit equation, telescoping argument, and signature mechanic are identical.

## Zcash Orchard (NU5, 2022)

Orchard preserves the binding-signature pattern with updates: Pallas/Vesta curves (cycle-friendly for Halo 2), no trusted setup, redesigned commitments. The cryptographic argument for value-balance is unchanged — the binding signature is still a Schnorr-style proof of knowledge of the value-balance discrete log.

## Mimblewimble (Grin, Beam, since 2018)

Mimblewimble's "kernel excess" is a per-tx point with a Schnorr signature, identical in spirit to Sapling's binding signature. MW additionally publishes a "kernel offset" scalar to enable transaction aggregation and cut-through unlinkability (a different concern from inflation prevention). Hathor does not aggregate transactions across authors at the network layer, so the kernel offset is not needed.

## Confidential Transactions (Liquid, original Maxwell proposal)

The original Confidential Transactions proposal by Greg Maxwell (2015) uses a per-tx Pedersen-commitment-balance signature with the same structure. Liquid (Blockstream's federated sidechain) implements this in production. The chain-wide supply audit equation is the same as proposed here, generalized over per-asset commitments.

## Monero RingCT

Monero's RingCT also uses a per-tx commitment-balance proof, integrated with the ring signature for spend authorization. The proof of value-balance is functionally a Schnorr-style DL knowledge proof; the difference is the integration with ring authorization.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Single-shielded-output shield-in policy.** Should consensus require at least 2 shielded outputs when all inputs are transparent? This RFC currently recommends but does not enforce. Arguments on both sides documented in [Section 4.5](#45-updated-transaction-rules) and [Rationale](#rationale-and-alternatives). Resolve before activation.

2. **`UnshieldBalanceHeader` removal lifecycle.** This RFC removes `UnshieldBalanceHeader` (`0x13`) and folds its role into the extended `ShieldedOutputsHeader`. Under migration Path A (co-activation), no legacy transactions exist. Under Path B (post-shielded hard-fork), pre-`H_audit` transactions retain `UnshieldBalanceHeader` in chain history, but it is rejected for new transactions after `H_audit`. The exact lifecycle for the existing `UnshieldBalanceHeader` code path (delete immediately, retain for verifying historical blocks only, etc.) needs explicit decision based on the chosen migration path.

3. **Asset balance binding signatures.** This RFC's binding signature covers value balance but not asset balance. For `FullShieldedOutput` transactions, asset blinding factors must also balance (`Σ s_inputs = Σ s_outputs`), enforced today by wallet construction analogous to the value blinding. Should we add a second binding signature for asset balance? Or rely on the existing surjection proof guarantees? Trade-off: one more 99 B header per shielded tx vs. weaker chain-wide asset audit.

4. **Per-token kernel excesses (HTR-only audit).** The current chain-wide equation is joint across all tokens. To audit HTR alone (independent of custom token issuer behavior), either constrain HTR to transparent operations only or publish per-asset kernel excesses (loses asset privacy). The right choice depends on HTR's deployment policy. Resolve before activation.

5. **Block-level binding signature aggregation.** Schnorr signatures within a block could be aggregated using MuSig-style techniques, reducing per-block cryptographic verification cost and storage. This is an optimization, not a correctness concern; defer to future RFC.

6. **Genesis-state initialization.** The audit equation requires a `total_supply[token]` initial value at genesis. For HTR, this is the publicly known initial allocation. For custom tokens, supply is zero at token creation and grows via mint operations. Specify the exact initial state and how it interacts with the snapshot offset under migration Path B.

7. **Sighash domain separator.** The signature's challenge hash uses domain separator `"HathorBindingSig/v1"`. Confirm this domain separator does not collide with any existing protocol hash (transaction sighash, ECDH key derivation, NUMS asset tag derivation).

# Future possibilities
[future-possibilities]: #future-possibilities

## Asset Balance Binding Signature

An analogous binding signature for asset blinding factors would enable chain-wide *asset* audit (in addition to value audit). The construction mirrors this RFC's value binding signature: publish `A_tx = a_tx · G` where `a_tx = Σ s_out − Σ s_in` (asset blindings), and sign with `A_tx` as pubkey. Cost: another 99 B header per shielded tx. Benefit: external observers can detect cross-asset forgery in the same way they detect inflation.

## Block-Level Binding Signature Aggregation

Multiple binding signatures within a block can be combined using MuSig or related multi-signature aggregation, reducing the per-block signature size from `64 · n` bytes to `64 + n_extra` bytes (where `n_extra` is small overhead). Verification also amortizes. Particularly valuable as shielded transaction volume grows.

## Cross-Block UTXO Pruning Invariants

The chain-wide audit equation depends only on `Σ C_utxo` and `Σ_tx E_tx`. If UTXO set commitments (e.g., utreexo or merkle-summed UTXO accumulators) are introduced, the audit equation can be incorporated into the accumulator, enabling stateless verification: a light client could verify supply without storing the entire UTXO set, given only the accumulator root and the running excess sum.

## Per-Token Public Audit Snapshots

Token issuers could publish periodic "audit snapshots" — signed claims about their token's supply at a given block height — verifiable against the audit equation. This combines the audit with issuer attestation for regulatory or transparency purposes.

## Bridging to Other Privacy Pools

If Hathor introduces a second shielded pool (e.g., an Orchard-style pool for stronger asset privacy), binding signatures from each pool combine homomorphically. Cross-pool transfers can be audited via a unified equation.

## HTR-Only Audit via Constrained HTR Operations

Independent of this RFC, a separate policy decision could constrain HTR mint, melt, shield, and unshield operations to be transparent, making HTR supply auditable from cleartext fields alone — independent of custom-token issuer behavior. The two changes compose: this RFC provides chain-wide auditability for all tokens jointly; the HTR-transparent policy provides HTR-specific auditability without depending on custom-token issuer correctness.

## Removal of Asset Privacy Tradeoff

Per-asset binding signatures (mentioned in Unresolved Question 4) leak which tokens appear in each transaction, weakening asset privacy. A future construction using zero-knowledge per-asset balance proofs (rather than per-asset binding signatures) could provide both per-asset chain-wide audit AND asset privacy. This is an open research direction; reasonable approaches include Halo 2 / Plonk-based asset-balance circuits, or Sapling's asset-aware extensions.
