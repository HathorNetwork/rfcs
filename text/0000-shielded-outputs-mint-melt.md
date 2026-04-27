- Feature Name: shielded_outputs_mint_melt
- Start Date: 2026-04-27
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Hathor Labs

# Summary
[summary]: #summary

This RFC extends [shielded outputs](./0000-shielded-outputs.md) to support **token
creation, mint, and melt** operations inside shielded transactions. It introduces two
new transaction headers — `MintHeader` and `MeltHeader` — that publicly declare the
per-token supply changes inside an otherwise-shielded transaction. The declared
amounts are bound into the homomorphic balance equation as public scalar terms,
allowing the verifier to enforce supply correctness without revealing where the
value lands. As a consequence, parent RFC Rule 8 ("Mint/Melt Transactions Cannot
Have Shielded Outputs") is replaced by the rules in this document, and
`TokenCreationTransaction` may now carry shielded outputs of the new token.

# Motivation
[motivation]: #motivation

The parent RFC delivers amount and token-type privacy for ordinary value transfers
but explicitly excludes mint and melt operations (parent Rule 8). Two real
problems follow:

- **Privacy continuity.** A token issuer who routinely transacts in shielded form
  must drop to fully-transparent mode to mint or melt, and then create a follow-up
  shielding transaction. This leaks the timing and economic shape of every
  issuance event, and breaks privacy continuity for the issuer's downstream
  business flows (payroll, vendor payments).
- **Confidential issuance.** Many issuance use cases — pre-funding salaries, B2B
  receivables, treasury operations — should reveal *the supply change* but not
  *the recipient set or the per-recipient amount*. Today, every minted unit is
  visible in plaintext outputs.

Auditability is preserved because the mint/melt amounts remain public per token —
declared in the new headers — so total supply per token is still publicly
computable. What becomes private is *where the new tokens land* and *which inputs
are melted from*.

**Expected outcome.** A shielded transaction can mint or melt any custom token
and create new tokens directly into shielded outputs, with the same auditability
guarantees as today's transparent mint/melt operations.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## 1. What changes

A shielded transaction may now exercise mint or melt authority. The amounts are
declared publicly via two new headers:

- **`MintHeader`** — list of `(token_index, amount)` entries declaring supply
  *created* in this transaction.
- **`MeltHeader`** — list of `(token_index, amount)` entries declaring supply
  *destroyed* in this transaction.

Both are required only for shielded transactions. Transparent-only mint/melt
transactions (no shielded inputs and no shielded outputs) continue to use the
existing implicit amount-balance equation — no header needed.

A given token may appear in **either** `MintHeader` **or** `MeltHeader` in a
single transaction, never both (see Rule M3).

Concrete worked examples for `DEPOSIT`-version tokens (with HTR deposit/withdraw
math) appear in §3.4 and §3.6; an example for `FEE`-version tokens (with
per-output fee math) appears in §3.5.

## 2. Token creation can now be shielded

Token creation transactions previously rejected shielded outputs outright. With
this RFC:

- The new token's UID is still `tx.hash` and occupies `tokens[0]` (token_index 1).
- Outputs of the new token may be transparent **or** shielded.
- The total initial supply is declared via a single `MintHeader` entry for the
  new token's index. The Pedersen balance equation reconciles transparent +
  shielded outputs against the declared supply.
- Mint/melt authority outputs for the new token remain transparent (parent
  RFC Rule 7).

## 3. Examples

### 3.1 Confidential mint

An issuer holds a transparent mint authority for token T and wants to mint
100,000 T directly into a single shielded output for a salary recipient.

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

Observers learn: *100,000 T were minted*. They do not learn the recipient or
that the issuer's HTR change is non-zero.

### 3.2 Confidential token creation

An issuer creates token T with initial supply 1,000,000, distributed across
4 shielded outputs.

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

The token UID is the tx hash; the supply (`1,000,000`) is publicly declared.
Per-recipient amounts are private.

### 3.3 Confidential melt

A treasurer melts 50,000 T (e.g., burning seized supply) without revealing
which UTXO was destroyed.

```
Inputs:
  - melt authority of T (transparent)
  - shielded T input (shielded)

Outputs:
  - melt authority of T (transparent, retained)
  - shielded HTR withdraw + change (shielded)

Headers:
  - FeeHeader
  - ShieldedOutputsHeader: 1 shielded output
  - MeltHeader: [(token_index=1, amount=50000)]
```

The melted amount is public; the input commitment is opaque on-chain.

### 3.4 DEPOSIT-version token: mint with HTR deposit

Token `TD` is `DEPOSIT`-version. The issuer mints 100,000 TD across two
shielded outputs. The Hathor 1% deposit rule applies: minting 100,000 TD costs
1,000 HTR. Assume the standard per-output fee is 1 HTR and shielded outputs
cost 1 HTR each (`FEE_PER_AMOUNT_SHIELDED_OUTPUT`).

```
Inputs:
  - mint authority of TD (transparent)
  - HTR (transparent), 1,002 HTR available

Outputs:
  - mint authority of TD (transparent, retained)
  - 2 shielded TD outputs (totals 100,000 TD)

Headers:
  - FeeHeader: 2 HTR fee (2 × FEE_PER_AMOUNT_SHIELDED_OUTPUT)
  - ShieldedOutputsHeader: 2 shielded outputs
  - MintHeader: [(token_index=1, amount=100000)]

Verifier:
  - deposit = 0.01 × 100,000 = 1,000 HTR (derived from MintHeader, applied
    as a public output term on H_HTR).
  - Augmented balance:
      sum(C_in) + 100,000·H_TD
        == sum(C_out) + 1,000·H_HTR + 2·H_HTR
  - HTR side reduces to: 1,002·H_HTR (input) == 1,000·H_HTR (deposit) + 2·H_HTR (fee).
  - TD side: 0·H_TD (no input of TD) + 100,000·H_TD == sum(shielded TD commitments).
```

What observers learn: 100,000 TD were minted; 1,000 HTR were burned for
deposit; 2 HTR fee was paid. They do not learn how the 100,000 TD were split
between the two shielded outputs or who the recipients are.

### 3.5 FEE-version token: mint paying per-output fees

Token `TF` is `FEE`-version. The issuer mints 50,000 TF into one transparent
output of 30,000 TF (paid to a public counterparty) plus one shielded output
of 20,000 TF (treasury). `FEE`-version tokens have no HTR deposit; per-output
fees apply instead.

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

Verifier:
  - No deposit/withdraw on HTR (FEE-version tokens skip the 1% rule).
  - Augmented balance:
      sum(C_in) + 50,000·H_TF
        == sum(C_out) + total_fee·H_HTR
    where the TF side balances 50,000 newly-minted units against
    30,000 (transparent) + 20,000 (shielded committed).
  - Total fee in FeeHeader must match exactly:
      FEE_PER_OUTPUT × 1 + FEE_PER_AMOUNT_SHIELDED_OUTPUT × 2.
```

What observers learn: 50,000 TF were minted; 30,000 TF went to a publicly
visible transparent output; the remaining 20,000 TF entered the shielded
pool. They do not learn the recipient of the shielded output or the issuer's
HTR change.

### 3.6 DEPOSIT-version token: melt with HTR withdraw

Token `TD` is `DEPOSIT`-version. The treasurer melts 80,000 TD; this releases
800 HTR back from the deposit pool to the spender (1% withdraw rule).

```
Inputs:
  - melt authority of TD (transparent)
  - shielded TD input (shielded, holds at least 80,000 TD plus optional change)

Outputs:
  - melt authority of TD (transparent, retained)
  - shielded HTR output (carries the 800 HTR withdraw + any change)
  - optional shielded TD change (if input held more than 80,000)

Headers:
  - FeeHeader: shielded fees only
  - ShieldedOutputsHeader
  - MeltHeader: [(token_index=1, amount=80000)]

Verifier:
  - withdraw = 0.01 × 80,000 = 800 HTR.
  - Augmented balance:
      sum(C_in) + 800·H_HTR
        == sum(C_out) + 80,000·H_TD + fee·H_HTR
  - TD side: input commitment holds X TD; output side holds (X − 80,000) TD
    (in optional change) plus 80,000·H_TD as a public melt term.
  - HTR side: 800·H_HTR (input from withdraw) is balanced by the recipient's
    shielded HTR output(s).
```

## 4. What remains visible

For every shielded mint/melt transaction, the following remains public:

- **Per-token supply delta**: each `(token_index, amount)` in MintHeader/MeltHeader.
- **Authority spend events**: mint/melt authority inputs and outputs are always
  transparent (parent Rule 7), so observers see *when* an authority is exercised.
- **HTR deposit/withdraw**: derived from the declared mint/melt amounts using
  Hathor's existing 1% deposit rule for `DEPOSIT`-version tokens.
- **Transaction structure**: input/output counts, transparent vs. shielded
  partition, and which outputs are authority outputs.

What becomes private:

- Recipient sets and per-recipient amounts of newly minted tokens.
- Which shielded UTXOs are consumed by a melt operation.
- The HTR-side change resulting from a deposit/withdraw, if shielded.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## 4.1 Header layout

### MintHeader

| Field | Size | Description |
|-------|------|-------------|
| Header ID | 1 B | `0x14` (`VertexHeaderId.MINT_HEADER`) |
| `num_entries` | 1 B | `1 ≤ num_entries ≤ 16` |
| Entries | variable | Concatenated entries |

Each entry:

| Field | Size | Description |
|-------|------|-------------|
| `token_index` | 1 B | `1 ≤ token_index ≤ len(tx.tokens)` (HTR/index 0 forbidden) |
| `amount` | 8 B BE | Public mint amount, `amount ≥ 1` |

Constraints:

- All `token_index` values within a header are distinct.
- All `amount` values are `≥ 1`. Zero entries are forbidden (zero entries leak no
  information and only inflate header size).
- `token_index = 0` (HTR) is forbidden — HTR is never minted by user
  transactions.

### MeltHeader

Wire format and constraints identical to `MintHeader`, with header ID `0x15`
(`VertexHeaderId.MELT_HEADER`). `token_index = 0` is similarly forbidden.

### Header order

Canonical ordering is ascending `VertexHeaderId`:

```
0x10 NANO_HEADER
0x11 FEE_HEADER
0x12 SHIELDED_OUTPUTS_HEADER
0x13 UNSHIELD_BALANCE_HEADER
0x14 MINT_HEADER
0x15 MELT_HEADER
```

`MintHeader` precedes `MeltHeader` when both are present.

### Maximum number of headers

`get_maximum_number_of_headers()` is raised from 3 to **at least 5**, to allow
`FeeHeader + (ShieldedOutputsHeader | UnshieldBalanceHeader) + MintHeader +
MeltHeader` plus a margin for future additions. The exact value is a consensus
parameter and is gated by feature activation.

### Sighash coverage

The full serialization of `MintHeader` and `MeltHeader` (header_id + count +
entries) is included in the transaction sighash. Mutating any entry invalidates
all signatures over the transaction. This is required because the declared
amounts directly affect the verified balance equation.

### Disclosure model

Each entry in `MintHeader` or `MeltHeader` discloses two things: the **token
reference** (`token_index`) and the **amount**. Their visibility differs.

**Token reference: always transparent.** The token reference is plaintext for
two reasons that together rule out hiding it:

1. The mint or melt authority output is itself transparent (parent Rule 7) and
   exposes the token UID via its `token_data`. Hiding the token in the header
   would leak nothing additional but provide no privacy benefit.
2. The verifier must read the token's version (`NATIVE` / `DEPOSIT` / `FEE`) to
   apply the correct deposit-or-fee logic. A hidden token would block this
   lookup, requiring an unbounded ZK proof of "I am applying the right rule for
   the right version".

**Amount: transparent in this RFC; shielded amounts are a pending business
decision.** The headers in this RFC carry plaintext `u64` amounts. The
trade-off is auditability vs. privacy:

- *Transparent amounts (this RFC's choice).* Total token supply is publicly
  computable per token by summing `MintHeader` and `MeltHeader` entries across
  the chain. Compatible with light-client supply auditors. **Required** for
  `DEPOSIT`-version tokens because the HTR deposit is `0.01 × amount` and must
  be verifiable against the public amount.
- *Shielded amounts (alternative, deferred).* The header carries a Pedersen
  commitment to the amount plus a range proof. Total supply becomes opaque on
  chain. Feasible for `FEE`-version tokens (fees are per-output, not
  per-amount). For `DEPOSIT`-version tokens this requires an additional ZK
  proof binding the HTR deposit to 1% of the committed amount, which is
  substantially more machinery than the rest of this RFC.

Because the choice has direct consequences for auditability — and because the
two token versions admit different complexity for shielded amounts — the RFC
treats this as a pending business decision (see [Unresolved
Questions](#unresolved-questions)).

## 4.2 Verification rules

Six new rules govern shielded mint/melt. All rules from the parent RFC
(Rules 1–7) continue to apply; **Rule 8 is replaced by Rules M1–M6 below**.

### Rule M1: Headers are valid only on shielded transactions

If `MintHeader` or `MeltHeader` is present, the transaction MUST be shielded
(`tx.is_shielded()`). Presence on a non-shielded transaction is rejected with
`HeaderNotSupported`.

### Rule M2: Mint/melt authority required for each entry

For each `(token_index, amount)` in `MintHeader`, the transaction MUST consume
at least one mint authority input for `tx.tokens[token_index − 1]`.

Symmetric for `MeltHeader` and melt authority. Authority inputs and outputs
remain transparent (parent Rule 7).

### Rule M3: One direction per token

For any given token, a single transaction may declare **either** a mint
**or** a melt, never both. Concretely: a `token_index` that appears in
`MintHeader` MUST NOT appear in `MeltHeader`, and vice versa.

Self-offsetting (mint X and melt Y of the same token in one tx) is meaningless
because the same net supply effect is achievable by adjusting amounts.
Permitting it would also complicate per-token supply accounting on the
indexer side without delivering any user-visible capability.

### Rule M4: Augmented homomorphic balance

The shielded balance equation (parent §4.4) is augmented:

```
sum(C_in) + sum_T(mint_T · H_T)  ==
    sum(C_out) + sum_T(melt_T · H_T) + sum(C_fee_entries) +
    deposit · H_HTR  −  withdraw · H_HTR
```

Where:

- For each `MintHeader` entry `(T, amount)`: add `amount · H_T` to the input
  side. The minted amount has no associated blinding factor (it appears
  unblinded in the balance equation).
- For each `MeltHeader` entry `(T, amount)`: add `amount · H_T` to the output
  side, also unblinded.
- `H_T = derive_asset_tag(token_uid_T)`.
- `deposit = Σ get_deposit_token_deposit_amount(amount)` over `MintHeader`
  entries whose token is `DEPOSIT`-version.
- `withdraw = Σ get_deposit_token_withdraw_amount(amount)` over `MeltHeader`
  entries whose token is `DEPOSIT`-version.

The `deposit` and `withdraw` terms move HTR through the equation in the same
way the existing transparent verifier handles them
(`_check_token_permissions_and_deposits`), but now sourced from the public
header amounts instead of inferred from transparent value flows.

`FEE`-version tokens contribute neither deposit nor withdraw (matching current
behavior).

### Rule M5: Trivial commitment protection still applies

Parent RFC Rule 4 (≥ 2 shielded outputs when all inputs are transparent, or
include a transparent output) is **not relaxed** by the presence of a
`MintHeader`. The minted amount enters the balance equation unblinded, so it
provides no entropy that would otherwise mask a single shielded output.

### Rule M6: FeeHeader still required

A shielded mint/melt transaction MUST carry a `FeeHeader` (parent Rule 2).
Deposit and withdraw amounts derived from `MintHeader`/`MeltHeader` are NOT
declared in the FeeHeader — they are folded into the balance equation
separately as Rule M4 specifies.

## 4.3 Surjection-domain extension

`FullShieldedOutput` requires a surjection proof showing its asset commitment
corresponds to one of the input domain generators (parent §4.1). For minted
tokens, no input contributes that asset, which would otherwise prevent
`FullShieldedOutput` of a freshly-minted token.

**Extension.** For each `(T, amount)` entry in `MintHeader`, the unblinded
NUMS asset tag `H_T = derive_asset_tag(token_uid_T)` is added to the surjection
proof domain. This permits a `FullShieldedOutput` of a minted token to prove
its asset is one of the (transparent inputs ∪ shielded inputs ∪ minted tokens).

`MeltHeader` does NOT extend the surjection domain, because melt produces no
new outputs of the melted token.

## 4.4 Token creation transactions

`TokenCreationTransaction` (TCT) gains the ability to carry shielded outputs.
The existing rejection
`InvalidShieldedOutputError('shielded outputs are not allowed in
TokenCreationTransaction')` is removed.

**TCT-specific rules.**

1. The new token UID is `tx.hash` and occupies `tokens[0]` (token_index 1) per
   existing semantics.
2. If the TCT is shielded (carries `ShieldedOutputsHeader`), it MUST carry a
   `MintHeader` whose entries include exactly one entry for `token_index = 1`,
   with `amount > 0`.
3. The single-entry-for-new-token rule replaces the existing
   `verify_minted_tokens` check (`token_info.amount > 0`) for shielded TCTs.
   Non-shielded TCTs retain the existing check.
4. The new token's `H_T` is added to the surjection-proof domain (per
   §4.3), enabling `FullShieldedOutput` of the new token.
5. Authority outputs for the new token remain transparent (parent Rule 7).

## 4.5 Verification pipeline integration

### Phase 1 (without storage)

- `verify_headers` (`vertex_verifier.py`): allow `MintHeader` and `MeltHeader`
  on `REGULAR_TRANSACTION` and `TOKEN_CREATION_TRANSACTION` when the
  `shielded_transactions` feature is active. Canonical-ordering check still
  applies.
- New `verify_mint_melt_headers_well_formed`:
  - Both headers, if present, are non-empty.
  - All `token_index` values within a header are unique.
  - No token appears in both headers (Rule M3).
  - All `token_index` values are in `[1, len(tx.tokens)]`.
  - All `amount` values are ≥ 1.
- New `verify_mint_melt_requires_shielded` (Rule M1).

### Phase 2 (with storage)

- New `verify_mint_melt_authority_inputs` (Rule M2): for each `MintHeader`
  entry, walk `tx.inputs`; at least one mint authority input must reference
  `tx.tokens[token_index − 1]`. Symmetric for melt.
- Updated `verify_shielded_balance` (Rule M4): incorporate the
  `MintHeader`/`MeltHeader` terms in the homomorphic balance call.
- Updated `_check_token_permissions_and_deposits` (shielded path): compute
  `deposit`/`withdraw` from `MintHeader`/`MeltHeader` entries instead of
  skipping them; fold into the HTR balance equation.
- Updated `verify_surjection_proofs` (§4.3): augment the domain with
  `derive_asset_tag(tokens[token_index − 1])` for each `MintHeader` entry.
- Removed: `verify_no_mint_melt`. Replaced by Rules M1–M2 + M4.
- Removed: TCT-specific blanket block on shielded outputs.

## 4.6 Wire format example

```
MintHeader serialization:
  header_id(1B=0x14) | num_entries(1B) |
  entry_0: token_index(1B) | amount(8B BE) |
  entry_1: token_index(1B) | amount(8B BE) | ...

Single-entry MintHeader for 100,000 of token at index 1:
  0x14 0x01 0x01 0x00 0x00 0x00 0x00 0x00 0x01 0x86 0xA0
```

`MeltHeader` is identical with `header_id = 0x15`.

## 4.7 Feature activation

The headers are gated by a sub-feature on top of the parent RFC's flag:

```python
class Feature(StrEnum):
    SHIELDED_TRANSACTIONS = 'SHIELDED_TRANSACTIONS'        # parent RFC
    SHIELDED_MINT_MELT = 'SHIELDED_MINT_MELT'              # this RFC
```

```python
ENABLE_SHIELDED_MINT_MELT: FeatureSetting = FeatureSetting.DISABLED
```

`SHIELDED_MINT_MELT` requires `SHIELDED_TRANSACTIONS` to be active (validated at
startup). When `SHIELDED_MINT_MELT` is disabled, parent RFC Rule 8 remains in
force and `MintHeader`/`MeltHeader` are rejected.

This phasing lets the parent RFC ship and stabilize before mint/melt support is
enabled, minimizing the blast radius if a verification bug is found.

## 4.8 Indexer and explorer impact

- **Token supply tracking** is fully restored: an indexer reconciles per-token
  supply by reading `MintHeader`/`MeltHeader` entries on every shielded tx in
  addition to existing transparent mint/melt detection.
- **Per-tx mint/melt event** is observable by anyone: `(token, amount,
  "mint" | "melt")` is published as plain header data.
- The "skip shielded inputs in rocksdb tokens index" pattern (commit `316e68b7`)
  extends naturally — shielded outputs of minted tokens are still skipped in
  per-UTXO indexing because their amount is hidden, but supply totals are
  derived from the headers.

# Drawbacks
[drawbacks]: #drawbacks

1. **Surface area growth.** Two new headers, new verifier paths, and a
   relaxation of parent RFC Rule 8. Each is security-sensitive.
2. **Header limit pressure.** Bumping `get_maximum_number_of_headers()` from 3
   to ≥ 5 is a consensus parameter change that must be carefully gated.
3. **Subtle balance equation.** Rule M4 changes the homomorphic balance
   equation. A bug here could enable inflation. Cross-checks against the
   existing transparent-deposit arithmetic are mandatory.
4. **Synthetic surjection-domain entries.** Adding minted-token asset tags to
   the surjection domain widens the anonymity set in a way the underlying
   library should already support, but it is a new code path that must be
   exercised in tests against `secp256k1-zkp`.
5. **Incomplete privacy.** Mint/melt amounts remain public per token. Issuers
   seeking *total* privacy of issuance are not served by this RFC.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why declare amounts publicly?

Pedersen commitments hide amounts but cannot, on their own, distinguish
"minted from nothing" from "received from an input of equal amount". Without a
public scalar declaring the mint, the prover could mint arbitrary amounts. The
public declaration binds the mint to a scalar that enters the balance equation,
preserving the Pedersen "no inflation" guarantee.

A purely private alternative — proving "I have authority to mint up to X and
am minting Y ≤ X" via a ZK proof — is an order of magnitude more complex
(circuits, trusted setup or larger proofs, new cryptographic dependencies) and
loses the auditability property. Public declaration is the lightest mechanism
that delivers the desired privacy improvement (recipients/inputs hidden) while
keeping supply auditable.

## Why a single MintHeader with a list (not multiple instances)?

The existing vertex header model enforces "at most one of each header type"
(`vertex_verifier.py:273-284`). Allowing multiple `MintHeader` instances would
require a special case. A single header carrying a list:

- Preserves the existing invariant.
- Enforces uniqueness across entries cheaply.
- Keeps the binary format compact.

## Why prohibit cross-token offsetting (Rule M3)?

Allowing the same token in both `MintHeader` and `MeltHeader` is meaningless:
the same net effect is achieved by adjusting amounts. Permitting it doubles
the verification surface for no benefit.

## Why not extend FeeHeader?

`FeeHeader` semantics are "transparent additions to the output side of the
balance equation, burned". Mint amounts are inputs (added to input side); melt
amounts are outputs but with deposit/withdraw side effects on HTR. Conflating
these into FeeHeader would obscure the equation and require the FeeHeader to
carry sign information.

## Why not a single combined `MintMeltHeader`?

Mint and melt have opposite signs in the balance equation and asymmetric
surjection-domain effects (mint extends the domain; melt does not). Separate
headers make the binary format and verifier state clearer, at the cost of one
extra header ID.

## Why not a MAC instead of a public amount?

A MAC of "this tx mints N of T" generated by the mint authority key adds a new
cryptographic dependency without strong benefit: the existing transaction
signature already authenticates the spend of the authority input. The amount
itself is what enters the balance equation.

## Impact of not doing this

Mint/melt operations remain forever transparent. Shielded users who need to
mint or melt drop to plaintext mode and create follow-up shielding transactions,
leaking timing and economic shape. Issuers seeking confidential issuance have
no on-chain option.

# Prior art
[prior-art]: #prior-art

## Liquid / Elements: Confidential Assets and Issuance

Elements supports confidential asset issuance in a single transaction. The
issuance carries a public asset_id and amount, very similar in spirit to
`MintHeader`. Elements additionally supports *blinded issuance*, the privacy
upgrade analogous to a future ZK-based version of this RFC.

## Zcash Sapling: value-balance field

Sapling tracks net flows between transparent and shielded pools via a
`valueBalance` field on the transaction. This is the closest analog to
declaring a public per-asset delta on an otherwise-shielded transaction.

## Monero

Monero has no token concept and therefore no direct analog for token mint/melt.
However, Monero's `RingCT` design demonstrates the soundness of using public
scalar amounts (in coinbase emission) inside a Pedersen-commitment balance
equation.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. **Sub-feature flag vs single flag.** Should `SHIELDED_MINT_MELT` be a
   distinct feature flag (allowing the parent RFC to ship first), or be folded
   into `SHIELDED_TRANSACTIONS` and shipped together?
2. **`get_maximum_number_of_headers()` value.** 5 is the minimum; 6 leaves
   margin for future headers but expands the consensus parameter further. Final
   value to be determined.
3. **`NanoHeader` interaction.** May a transaction simultaneously carry a
   `NanoHeader` and be a shielded mint/melt? Phase-1 simplification: forbid
   the combination; reconsider when shielded Nano Contracts are designed
   (parent RFC §4.8).
4. **Transparent vs. shielded mint/melt amounts.** This RFC declares amounts
   as plaintext `u64` (see [Disclosure model](#disclosure-model)). The
   alternative — Pedersen commitments with range proofs — would hide token
   supply entirely from public observers, at the cost of: (a) substantial
   extra ZK machinery for `DEPOSIT`-version tokens (a proof that the HTR
   deposit equals 1% of the committed amount); (b) loss of the public
   "supply auditor" property that lets any node compute total supply per
   token; (c) divergent behavior between `DEPOSIT`- and `FEE`-version tokens
   (the latter is straightforward to shield since fees are per-output). The
   choice is a business decision about how much auditability to trade away.
   This RFC's plaintext design can be retrofitted later via a new feature
   flag without invalidating already-issued tokens.
5. **Authority-output presence requirement.** Rule M2 requires an authority
   *input* but does not require the transaction to produce a corresponding
   authority *output*. Should we additionally require authority retention by
   default to prevent accidental authority burn? (Current Hathor behavior
   permits intentional authority destruction, so no change recommended.)
6. **Shielded UnshieldBalanceHeader interaction.** A full-unshield transaction
   that also mints (e.g., the issuer mints into a transparent recipient)
   carries `UnshieldBalanceHeader + MintHeader`. The combined balance equation
   is well-defined but warrants explicit test coverage.

# Future possibilities
[future-possibilities]: #future-possibilities

## Blinded mint amounts

Extend `MintHeader` to optionally carry a Pedersen commitment instead of a
plaintext amount, with a range proof and a "mint quota proof" demonstrating
the mint stays within an authority-bound limit. Requires a quota mechanism
attached to mint authorities.

## Mint to specific recipients

A future header could bind a mint event to a specific destination, useful for
compliance-aware tokens (e.g., regulated stablecoins where issuance must be
attributable to a registered counterparty).

## Cross-token atomic batching

Multi-token support in `MintHeader` already allows atomic multi-token
issuance. A future RFC could formalize a "token bundle" semantic on top of
this primitive (e.g., simultaneously minting a governance token and its
matching reward token).

## Confidential authority provenance

Combine with Phase C (input unlinkability): the mint authority input itself
could be obscured via ring signatures, hiding *which authority UTXO* was
exercised even when the authority is owned across multiple keys.
