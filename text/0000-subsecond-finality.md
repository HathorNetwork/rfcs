- Feature Name: subsecond_finality
- Start Date: 2026-06-16
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Marcelo Salhab Brogliato <msbrogli@hathor.network>

# Summary
[summary]: #summary

This RFC specifies **sub-second finality** for Hathor: a fast *soft-finality* tier, produced by a fixed
committee of **finality validators**, layered on top of the existing merged-mined Proof-of-Work
*hard-finality* tier. Each validator, the first time it sees a transaction spending a given UTXO,
**pins** that UTXO to the transaction and co-signs it with a BLS signature — and will never sign a
conflicting spender. When validators of total weight `≥ 2f+1` (out of a committee calibrated as
`W = 3f+1`) have signed, their votes aggregate into a single, constant-size **Finality Certificate
(FC)**. A transaction is *softly final* the moment an FC exists for it — typically well under a second —
and *hard final* the moment a PoW block settles it. The two tiers are bound by one consensus rule: **a
PoW block is invalid if it confirms a transaction that conflicts with an already-certified one**, so PoW
*ratifies* the fast path and can never silently reverse a soft-final payment. This document describes the
v1 implementation as built in `hathor/finality/`: the UTXO fast path on a fixed PoA-style committee, with
a *certified-only mempool* and *validator-driven* certificate collection.

# Motivation
[motivation]: #motivation

Hathor settles transactions through merged-mined PoW blocks, which gives it Bitcoin-grade objective
security but inherits PoW's probabilistic, multi-second confirmation latency. For the payment use cases
Hathor is pursuing (point-of-sale, stablecoin rails), that latency is the single most visible gap against
card networks and modern payment-oriented chains that advertise sub-second confirmation.

Simply making blocks faster trades away security (shorter blocks mean lower per-block work and more
reorgs) and does not change the fact that PoW finality is *probabilistic*. The design space payment chains
have converged on instead keeps a slow, objective base layer and adds a **fast validator layer on top**
that provides an immediate, cryptographically verifiable confirmation (Dash InstantSend, Sui owned
objects, the Thunderella line of work).

The expected outcome:

- **Sub-second soft finality** for ordinary UTXO payments — enough for a merchant to release goods,
  comparable to a card authorization.
- **Unchanged hard finality**: the same merged-mined PoW settlement Hathor has today, now *ratifying* the
  fast path rather than competing with it.
- **No new trusted operator that can steal or forge state.** The validators can, at worst, *stall* (a
  liveness failure); they can never reverse a settled payment or mint value. A safety violation requires a
  collusion that is, by construction, evidenced by two contradictory signatures over the same UTXO.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## New concepts

- **Finality validator.** A node in a fixed, network-configured committee that co-signs UTXO transactions.
  Each validator has a **voting weight**. The committee and the weights live in the network settings
  (`FINALITY` in `hathor/conf/settings.py`), PoA-style, and are gated behind the `TWO_TIER_FINALITY`
  feature-activation bit, so every node knows who the validators are and when the subsystem turns on.
- **The pin.** The first time a validator sees a transaction spending a UTXO, it records `(tx_id, index)
  → spender_tx_id` in its **pin store** and signs the transaction. The pin is *immutable*: the validator
  will never sign a different transaction spending that UTXO. This is the only rule a validator must
  follow for safety.
- **Vote.** A single validator's BLS signature over a transaction's canonical *pin-message*
  (`Vote` in `hathor/finality/fc.py`).
- **Finality Certificate (FC).** An aggregate of a quorum of votes: one 96-byte BLS signature plus a
  committee bitmap, proving validators of weight `≥ 2f+1` pinned the transaction
  (`FinalityCertificate` in `hathor/finality/fc.py`). It verifies against the committee with a single
  `FastAggregateVerify` pairing check.
- **Soft finality.** A transaction is *softly final* once an FC exists for it. This holds under the
  assumption that the validator quorum is honest.
- **Hard finality.** A transaction is *hard final* once a merged-mined PoW block settles it — the
  objective, Bitcoin-grade tier Hathor has today.
- **Ratification rule.** A PoW block is **invalid** (and gets voided) if it confirms a transaction that
  conflicts with a transaction that already holds an FC. PoW therefore *ratifies* the fast path and can
  never silently reverse it.

## What is eligible

The fast path serves **single-owner UTXO transactions**: a non-nano `Transaction` with at least one input
(`FinalityService.is_eligible`). Blocks, genesis, and Nano Contract transactions (which touch shared
mutable state and need a total order) do not take this path; they follow the existing flow unchanged.

## How a payment flows

Take Alice paying Bob:

1. **Submit.** Alice's node creates `T` and, because the feature is active and `T` is eligible, the node
   does *not* place `T` in the mempool. It keeps `T` in an in-memory **pending pool** and forwards it to a
   committee validator. Pending transactions are never relayed to the general network by non-validators.
2. **Collect.** Validators gossip `T` among themselves. Each honest validator checks `T`, pins every one
   of its inputs, signs the pin-message, floods its `VOTE`, and accumulates incoming votes. The first
   validator to accumulate weight `≥ 2f+1` assembles the FC.
3. **Release.** That validator broadcasts the *certified transaction + FC* to the entire network. Every
   node independently re-verifies the FC reaches quorum, stores it, admits `T` to its mempool, and relays
   it onward. Bob's node sees the FC — the payment is **softly final**, about a quarter to half a second
   after Alice tapped.
4. **Settle.** A PoW block later confirms `T`; it is now **hard final**. Nothing Bob can observe changes,
   but the payment is now objectively irreversible.

The crucial discipline for receivers: **"final" means "the FC exists," not "a validator told me it was
ok."** A single validator's acknowledgement is not finality; only the quorum certificate is.

## Why a quorum is safe: every two certificates share an honest validator

The whole design rests on a single counting fact. The committee has total weight `W = 3f + 1` and
tolerates at most `f` weight of **rogue** (Byzantine, equivocating) validators. A quorum is any set of
weight `≥ 2f + 1`. Take any two quorums `Q1` and `Q2`. Their combined weight is at least
`(2f + 1) + (2f + 1) = 4f + 2`, but the whole committee is only `3f + 1`, so by inclusion–exclusion they
must **overlap in weight at least `(4f + 2) − (3f + 1) = f + 1`**. Since at most `f` weight is rogue, that
shared overlap of weight `f + 1` always contains **at least one honest validator**.

That honest validator is the linchpin. It pins each UTXO immutably and signs at most one spender of it.
So it can belong to a quorum for `T1` *or* a quorum for `T2`, never both. Therefore **two conflicting
transactions can never both reach a quorum** — at most one finality certificate can ever exist for a given
UTXO.

A concrete committee makes this tangible. Take `f = 1`: `W = 3f + 1 = 4` validators (`v1..v4`, weight 1
each), quorum `2f + 1 = 3`, tolerating one rogue validator. Any two 3-of-4 quorums overlap in at least
`f + 1 = 2` validators; with at most one rogue, at least one of those two is honest. The examples below all
use this committee.

## Example (i): no conflict

Alice creates `T` spending UTXO `x` to Bob and submits it. Every honest validator sees `x` unspent and
unpinned, so each pins `x → T` and signs. `T` collects all 4 votes — well past the quorum of 3 — and an FC
is assembled and broadcast. Bob's node verifies the FC and the payment is softly final in a fraction of a
second. This is the overwhelmingly common case: an honest, non-conflicting payment is certified by the full
committee almost immediately.

## Example (ii): a conflict that resolves

Alice tries to double-spend `x`: she sends `T1` (paying Bob) to validators `v1, v2, v3` and `T2` (paying
Carol) to `v4`, and the race plays out so that `v1, v2, v3` each pin `x → T1` and sign `T1`, while `v4`
pins `x → T2` and signs `T2`.

- `T1` gathers votes from `v1, v2, v3` — weight 3, a quorum. Its FC is assembled and broadcast. **Bob is
  paid; `T1` is softly final.**
- `T2` now has only `v4` (weight 1). For `T2` to also reach quorum it would need 3 signers, but `v1, v2, v3`
  have each *immutably* pinned `x → T1` and will never sign `T2`. The only validators left for `T2` are
  rogue ones, and there is at most `f = 1` of those. **`T2` can never reach quorum**, so no second FC is
  possible. Carol's node never sees an FC for `T2`, so it never treats the payment as final.

The quorum-intersection fact is exactly what forbids the second certificate: any quorum for `T2` would have
to share an honest validator with `T1`'s quorum, and that validator already pinned `x → T1`.

## Example (iii): a conflict that gets stuck

Alice splits the committee evenly: `T1` (to Bob) reaches `v1, v2`, and `T2` (to Carol) reaches `v3, v4`.
Each honest validator pins whichever it saw first, so `T1` collects weight 2 and `T2` collects weight 2 —
**neither reaches the quorum of 3.** No FC forms for either, and the UTXO `x` is now **stuck (wedged)**:
it produces no soft finality at all until a PoW block eventually settles one of the spenders.

This harms only Alice. Bob and Carol were never at risk, because each waits for an FC and neither ever
formed. Honest users never equivocate, so an honest user's UTXO never gets stuck — the wedged case is
exclusively self-inflicted by an equivocator.

### Resolving a stuck UTXO is an open question

What v1 does today is the conservative thing: a wedged UTXO simply stays frozen on the fast tier, and the
slow tier resolves it — a PoW block eventually confirms exactly one of the conflicting spenders (Hathor's
existing consensus already picks a single winner among conflicts), at which point the validators release
their now-moot pins. That is always safe, but the UTXO loses its sub-second finality until a block arrives.

How to recover *soft* finality for a stuck UTXO faster is **not settled in this design**. The common
approaches, each with trade-offs, are:

- **PoW tie-break.** Let the eventual block's choice be the canonical resolution and reset the pins to it.
  Simple and already implied by the slow tier, but it offers no *fast* recovery — it is just the frozen
  case named.
- **Epoch-boundary unlock / forgiveness** (as in Sui Lutris): at committee rotation, forgive equivocated
  pins so the UTXO can be re-attempted in the next epoch. Recovers liveness without weakening safety, at
  the cost of an epoch's delay and a rotation mechanism Hathor does not yet have.
- **Timeouts / refundable locks**, especially for multi-owner spends, so a co-signer cannot wedge a joint
  transaction indefinitely.

These are recorded as future work (see *Unresolved questions* and *Future possibilities*); v1 deliberately
ships only the always-safe "stays frozen until PoW" behaviour.

## Two layers of security, one of them battle-tested

It is worth being explicit about what the validator committee is and is not trusted for. There are **two
independent layers** guarding against a double-spend:

1. **The fast (finality) layer** — the validators, pins, and BLS certificates described above. Its safety
   rests on the honest-quorum assumption *and* on this code being correct.
2. **The base consensus layer** — Hathor's existing merged-mined PoW consensus, which already guarantees
   that among any set of conflicting transactions **at most one is ever confirmed** in the best chain. This
   layer is years old and battle-tested, and it is completely independent of the finality code.

The key consequence: **even if the finality implementation had a bug** that somehow produced two
certificates for conflicting spenders of the same UTXO, the base consensus layer would *still* confirm only
one of them — the other would be voided as a conflict, exactly as it is today without finality. A finality
bug can therefore, at worst, **break the soft-finality promise** (a payment believed final is reversed, or
a UTXO is wedged); it can **never cause an actual double-spend to settle on-chain**, because the ledger of
record is the PoW chain, not the certificate.

The ratification rule (a block may not confirm a transaction conflicting with a certified one) tightens
this further by forcing PoW to confirm the *certified* winner rather than an arbitrary one — but the raw
no-double-spend guarantee does not depend on the finality layer being correct at all. The fast tier buys
speed; the consensus tier remains the ground truth for safety.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation lives in the `hathor/finality/` package. Network and storage effects are reached
through an injected `FinalityTransport` and callbacks, so the protocol logic in `FinalityService` is
independent of the p2p and storage machinery and is unit-testable in isolation.

```
 any node (submitter)        committee overlay (validators gossip among themselves)        whole network
 --------------------        -----------------------------------------------------         -------------
 pending pool
   └─ forward to a  ──────▶  v_i: validate + PIN every input + sign ──flood tx + VOTE──▶ v_j, v_k …
      validator              each validator accumulates votes in its pending pool
   (await certified result)  first to reach weight ≥ 2f+1 → assemble FC
                             broadcast certified tx + FC ─────────────────────────────▶ every node:
                                                                                        verify FC quorum,
                                                                                        store FC, admit to
                                                                                        mempool, relay,
                                                                                        ratify in consensus
```

## Cryptography (`crypto.py`)

Validators co-sign with **BLS12-381** signatures (G2 proof-of-possession ciphersuite,
min-pubkey-size layout: public key = 48 bytes in G1, signature = 96 bytes in G2). Because every validator
signs the *same* per-transaction message, votes aggregate with same-message aggregation and an FC verifies
with `FastAggregateVerify` — one multi-pairing check whose cost and output size are **constant in committee
size**.

Same-message aggregate verification is only safe against rogue-key attacks when every committee public key
carries a verified **proof-of-possession (PoP)**; the PoP is checked when the committee is loaded
(`FinalitySettings._validate_and_derive`) and when a signer key file is loaded
(`FinalityValidatorSignerFile`).

The canonical message a validator signs is:

```
get_pin_message(tx_id, committee_hash) = sha256( b"hathor-fc-pin-v1" || committee_hash || tx_id )
```

It commits to `tx_id` **only**. This is sound because an honest validator pins *every* input of the
transaction before signing, so the pinned set is fully re-derivable from the transaction itself during
certificate verification. The committee hash binds a signature to a specific committee, and the domain tag
(`PIN_MESSAGE_TAG`) prevents cross-protocol replay.

The backend is the native **`blst`** library, reached through the project's Rust extension (`htr_lib`)
and wrapped behind module-level functions
(`bls_keygen / sk_to_pk / pop_prove / pop_verify / sign / verify / aggregate / fast_aggregate_verify`),
so it stays swappable without touching the rest of the codebase. Ordinary signatures and
proofs-of-possession use **distinct domain-separation tags** (`BLS_SIG_..._POP_` vs `BLS_POP_..._POP_`),
so a PoP can never be replayed as an ordinary vote. Because keys and signatures are parsed from untrusted
network data, verification keeps subgroup checks (`sig_groupcheck` / `pk_validate`) **on** and treats every
malformed input as a verification failure rather than an error.

Native crypto is what makes the fast path viable: a benchmark
([`0000-subsecond-finality-bls-benchmark.md`](0000-subsecond-finality-bls-benchmark.md)) measured `blst` at
≈ 0.6 ms per certificate verify — < 0.5% of the ~150–300 ms network round-trip and **constant in committee
size**. A pure-Python BLS reference (`py_ecc`) was ~400× slower (≈ 240 ms per verify, ≈ 16 s to verify a
100-vote quorum), which would have blown the sub-second target on its own.

`FinalityValidatorSigner` holds a runtime private key and exposes `sign_pin`; `FinalityValidatorSignerFile`
is the pydantic key-file (hex private key, public key, PoP) that validates internal consistency on load.
Both mirror the existing `PoaSigner` / `PoaSignerFile`.

## Committee configuration (`finality_settings.py`)

`FinalitySettings(enabled, validators[])` carries the committee. Each `FinalityValidatorSettings` is a
`(public_key, pop, weight)` triple. On load, when enabled, it:

- rejects an empty committee and duplicate public keys,
- verifies each PoP,
- derives total weight `W`, calibrates `f = (W - 1) // 3` (the largest `f` with `3f + 1 ≤ W`), and sets
  `quorum_threshold = 2f + 1`.

Quorum is **weight-based, not count-based**: `reaches_quorum(bitmap)` sums the weights of the validators
selected by a committee bitmap and compares to `2f + 1`. `calculate_committee_hash()` is a stable SHA-256
over the validators' JSON, used both to bind votes (in the pin-message) and to gate peer connections.

These settings are wired into `HathorSettings` as `FINALITY`, alongside the hard flag
`ENABLE_TWO_TIER_FINALITY`, the `CAPABILITY_FINALITY` p2p capability string, and the
`Feature.TWO_TIER_FINALITY` activation bit (`Features.two_tier_finality`).

## Wire objects (`fc.py`)

- **`Vote(tx_id, validator_id, signature)`** — `tx_id(32) || validator_id(2) || signature(96)` = 130
  bytes. The `validator_id` is a non-unique 2-byte skip-hint (first bytes of the hashed public key,
  mirroring `PoaSignerId`); it is only used to find a candidate public key quickly and is **never** trusted
  for membership — the signature is always verified against the committee's actual keys.
- **`FinalityCertificate(tx_id, bitmap, agg_signature)`** —
  `tx_id(32) || bitmap_len(1) || bitmap(bitmap_len) || agg_signature(96)`. The bitmap is a big-endian
  integer; bit `i` set means the validator at committee index `i` is in the quorum.
  `verify(settings)` returns true iff `reaches_quorum(bitmap)` **and** the aggregate signature verifies via
  `bls_fast_aggregate_verify` over the recomputed pin-message. Both objects fit comfortably in a single
  p2p line (limit 65536 bytes).

## Persistent stores (`stores.py`)

Two stores are **authoritative**, not derived from the DAG, and therefore cannot be rebuilt by scanning
it. They are deliberately kept out of `IndexesManager.iter_all_indexes()` so a reindex or
`--reset-indexes` can never wipe them, and they are exposed as `indexes.finality_pin` /
`indexes.finality_certificate`:

- **Pin store** (`FinalityPinStore`): `(tx_id, index) → pinned spender tx_id`. Its `try_pin` pins a UTXO
  to a spender iff it is unpinned or already pinned to the same spender, and returns `False` on a conflict
  (the caller would be equivocating and must not sign). Pins are immutable. `unpin_resolved` releases pins
  after settlement (housekeeping only). **Losing this store would let a validator sign a second,
  conflicting spender after a restart — the single highest-severity failure**, which is why it is never
  rebuildable/clearable.
- **Certificate store** (`FinalityCertificateStore`): `tx_id → FC bytes` for every FC a node has verified
  and accepted. The ratification rule consults it with O(1) `has_certificate` lookups.

Each has a RocksDB implementation (own column family) and an in-memory implementation for tests.

## Pending pool (`pending_pool.py`)

`PendingFinalityPool` holds finality-eligible transactions that do not yet have a quorum FC, keyed by
`tx_id`, each with the votes accumulated so far (by committee index). It is **in-memory**: its contents are
pre-final and can always be re-collected, so they need not survive a restart. `add_vote` is idempotent per
validator index (anti-double-count). `is_ready` / `assemble_certificate` aggregate the accumulated votes
(in committee-index order, so the bitmap and the aggregate agree) once they reach quorum.

## The service (`service.py`)

`FinalityService` orchestrates the fast path on every node. A node is a **validator** iff it was
constructed with both a signer and a pin store (the constructor asserts they are provided together). The
key entry points:

- **`should_divert_to_pending(vertex)`** — the mempool gate predicate: true for a finality-eligible
  transaction, while the feature is active, that does not yet have a known certificate.
- **`submit_local_transaction(tx)`** — for a locally-created transaction: keep it pending and either vote
  on it directly (if this node is a validator) or `submit_to_validator`.
- **`on_submit_finality_tx(tx, source)`** — a pending tx received from a peer. A non-validator just
  forwards it to a validator; a validator handles it.
- **`_handle_as_validator`** — floods the tx to other validators (if newly seen), then applies the voting
  rule and, if it passes, ingests its own vote.
- **`_try_vote(tx)`** — the **voting rule**. Returns `None` (no vote) if the tx already has a certificate,
  is double-spending, or spends a voided tx; if a dependency is not yet known (`TransactionDoesNotExist`),
  it **defers** (the parent's certificate may still arrive). Otherwise it first checks no input is already
  pinned to a *different* transaction, then **atomically pins every input** to this transaction (aborting
  without signing on any conflict), and finally signs the pin-message and returns a `Vote`.
- **`on_vote` / `_ingest_vote`** — verify a vote (resolving its signer by trying each committee key whose
  2-byte hint matches and checking the BLS signature), record it, re-flood it, and — if it completes a
  quorum — `_certify`.
- **`on_certificate` / `_accept_certificate`** — **independent admission**: every node re-runs
  `certificate.verify(settings)` before storing the FC, removing the tx from the pending pool, admitting it
  to the mempool (via the injected `admit_certified_tx` callback), and re-broadcasting. A node never trusts
  an upstream peer's certificate.
- **`on_settlement`** — housekeeping: once a transaction is hard-settled by a block (`first_block` set and
  not voided), release the validator's pins for its inputs. Safety does not depend on this (a consumed
  UTXO can never be re-spent); it frees pin storage and is the hook for future wedge recovery.

## p2p transport (`transport.py`, `messages.py`, `states/ready.py`)

`P2PFinalityTransport` implements the `FinalityTransport` protocol over existing peer connections. The
"committee overlay" is, in v1, simply the set of connected peers advertising `CAPABILITY_FINALITY`. Vote
authenticity rests on BLS verification against the committee — **not** on peer identity — so a
non-validator peer can neither forge a vote nor reach a quorum. Three new messages (gated by the shared
capability) are added in `ProtocolMessages` and handled in `ReadyState`:

- `SUBMIT_FINALITY_TX` — any node → a validator: the full pending transaction (hex). `submit_to_validator`
  picks the first available finality peer deterministically; validator gossip fans it out.
- `FINALITY_VOTE` — validator ↔ validator flood: a `Vote`.
- `FINALITY_CERTIFICATE` — broadcast to all peers: a JSON `{tx, fc}` payload carrying the certified
  transaction and its certificate (the FC travels *alongside* the tx because it signs over `tx_id` and
  cannot live inside the tx hash).

Handlers deserialize defensively, closing the connection on malformed input. Because mismatched committees
must never exchange finality data, `FINALITY` is mixed into the peer-hello settings hash
(`get_settings_hello_dict`) when enabled, mirroring `PoaSettings`.

## Certified-only mempool gate (`vertex_handler.py`)

The `VertexHandler` is the single funnel that guarantees the mempool only ever holds FC-backed finality
transactions. In both `on_new_mempool_transaction` and `on_new_relayed_vertex`, `_divert_to_finality`
checks `should_divert_to_pending`; if true, it hands the tx to `on_submit_finality_tx` (which keeps it
pending and gets it certified) and **returns without admitting or relaying it**. A certified transaction
arriving via `FINALITY_CERTIFICATE` re-enters through `admit_certified_tx` after independent verification.
The service is set after construction (`set_finality_service`) to avoid a circular dependency, and is
`None` when the feature is disabled.

## Ratification rule (`block_consensus.py`)

The binding rule lives in consensus, not the verifier, because the confirmed-transaction set only exists at
consensus time. In `BlockConsensusAlgorithm._score_block_dfs`, just before a block becomes best chain (and
sets `meta.first_block`), the algorithm calls:

- **`_fc_enabled_for(block)`** — gates on the block parent's `Features` (mirroring `_should_execute_nano`),
  false for genesis.
- **`_confirms_certified_conflict(block)`** — for each transaction the block confirms, and each input, it
  scans the conflicting siblings (`spent_meta.spent_outputs[index]`); if any sibling `t' != tx` holds a
  certificate (`fc_index.has_certificate`), the block is confirming a transaction conflicting with a
  certified one.

When that holds, the block is **voided** by reusing the existing non-winner voiding path (`must_void`) —
it becomes a voided side block while the certified tx and its descendants stay valid. This composes with
reorg handling for free and never crashes the node. The rule gates on **FC existence**, not on
`first_block`, because a certified transaction need not yet be block-confirmed.

## Wiring

`--finality-signer-file` (in `run_node_args.py`) supplies a validator's key. Both `Builder` and
`CliBuilder` construct the `FinalityService` (with the pending pool, certificate store, transport, and
admit callback) on every node when the feature is enabled, and additionally load the signer and pin store
on validator nodes. Settlement pin-release is driven by a pubsub adapter (`handle_consensus_event` →
`on_settlement`).

## Safety argument

The safety property — **two conflicting transactions can never both obtain an FC** — follows from quorum
intersection plus immutable pins. Any FC needs validators of weight `≥ 2f+1`. Two FCs over conflicting
spenders of the same UTXO would need two such quorums; in a committee of total weight `W = 3f + 1`, any two
weight-`(2f+1)` sets intersect in weight `≥ f + 1`, i.e. at least one honest validator (weight `> f` of
honest weight). But an honest validator pins each UTXO immutably and signs at most one spender, so it
cannot be in both quorums. Therefore at most one conflicting spender is ever certified. A dishonest
validator that signs both leaves two contradictory signatures over the same UTXO — objective, attributable
evidence of equivocation (the basis for future slashing). Validators can withhold signatures and **stall**
a UTXO (a liveness failure, recoverable by the PoW tier), but they can never reverse or forge.

## Latency

An end-to-end benchmark
([`0000-subsecond-finality-latency-benchmark.md`](0000-subsecond-finality-latency-benchmark.md)) drives
the real `FinalityService` fast path — pin, sign, flood the vote, accumulate, assemble the certificate,
verify, admit — over a discrete-event simulated network whose per-hop latency `L` is a parameter and whose
handler CPU is the *measured* `blst` cost. Two findings shape the design:

- **Soft finality ≈ `2·L + ~5 ms`.** A quorum is collected in two flood hops (the transaction out to the
  committee, the votes back), plus a few milliseconds of crypto on the critical path. The network
  round-trip dominates; sub-second holds with large margin for any realistic inter-validator latency
  (a 100-validator committee on 100 ms links reaches soft finality in ~205 ms).
- **Flat in committee size.** The node-local floor stays ≈ 3–5 ms from `N = 4` to `N = 100`, because
  `FastAggregateVerify` is constant in `N` and per-vote verification is ~1 ms. The committee can grow for
  decentralization at essentially no latency cost — but note vote gossip is all-to-all, so a validator's
  *work per transaction* still grows with `N` (`O(N)` vote verifies), even though the *latency* to first
  certification does not.

This is the measured basis for the "sub-second" claim, and it is why the design tolerates a sizable
committee: the latency budget is spent almost entirely on the network, not on validators or crypto.

# Drawbacks
[drawbacks]: #drawbacks

- **Soft finality is a weaker guarantee than PoW.** It rests on the honesty of a `2f+1` validator quorum.
  Although accountable, it is weaker than objective hashpower, and receivers who treat soft finality as
  equivalent to settlement take on that assumption. Clear UX is required.
- **Validators are a new locus of centralisation and censorship.** A committee can delay (censor)
  transactions even though it cannot reverse them. The escape valve is the PoW tier, but censorship
  resistance is weaker than a pure-PoW mempool.
- **New liveness corner cases.** Wedged UTXOs, partitions, and multi-owner griefing are new failure modes.
  They are safe (self-harm only) but add complexity, and in v1 a wedged UTXO simply stays frozen until PoW
  resolves it.
- **Operational complexity and layer coupling.** Block validity now depends on FCs, coupling two
  previously independent layers. Validator-set governance, key rotation, and FC dissemination are new
  responsibilities.
- **A native crypto dependency.** Sub-second verification relies on the native `blst` backend (via the
  `htr_lib` Rust extension) — a pure-Python BLS would be ~400× too slow (≈ 240 ms per verify). This is a
  compiled dependency in the hot path, though one already carried by the project's Rust crate.
- **Per-validator work grows with committee size.** Vote gossip is all-to-all, so each validator performs
  `O(N)` vote verifications per transaction and the committee does `O(N²)` total. Soft-finality *latency*
  stays flat in `N` (see *Latency*), but validator CPU and bandwidth do not, which bounds how large the
  committee can grow before vote dissemination needs a smarter (e.g. aggregation-tree) overlay.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The core choice is *how the fast tier resolves conflicts*:

1. **Single leader / sequencer per epoch** (Solana, rollups). One party orders everything so conflicts
   cannot arise — but it is a single point of liveness and censorship, and leader failover needs BFT
   anyway. Rejected: it reintroduces exactly the privileged operator Hathor's value proposition argues
   against.
2. **Consensusless quorum certificates** (this RFC; FastPay / Sui owned objects / Dash InstantSend).
   Leaderless, one round-trip, no total order, and a structural fit for single-owner UTXOs. Its only
   weakness — wedged UTXOs — is covered by the PoW tier Hathor already has. **Adopted.**
3. **Full total-order BFT for everything** (Tendermint, HotStuff). Strong but pays consensus latency for
   *every* payment, including the vast majority that never conflict. Rejected for the UTXO path; reserved
   for the Nano-Contract path, where shared state genuinely needs a total order.

The **certificate-as-block-validity-rule** is the load-bearing decision that makes the two tiers coherent:
it turns "PoW could in principle reorg the fast path" into "PoW provably ratifies it." Without it, soft and
hard finality could disagree and the soft tier would be merely advisory.

Implementation-level choices and the rationale for each:

- **BLS over Schnorr.** Both are orders of magnitude under the network latency budget, so raw speed is not
  the differentiator. BLS is chosen for *structure*: non-interactive observe-then-aggregate, constant-size
  and accountable certificates, and no nonce state — all of which fit a leaderless committee. (Benchmarks:
  [`0000-subsecond-finality-bls-benchmark.md`](0000-subsecond-finality-bls-benchmark.md).)
- **Weight-based `3f+1` calibration** rather than node counts, so the committee can use stake- or
  trust-weighted voting without changing the quorum math.
- **`tx_id`-only pin-message** (rather than committing to the input set in the signature), sound because
  honest validators pin every input before signing — keeping votes and certificates compact and
  re-derivable.
- **Certified-only mempool + validator-driven collection.** This deviates from the original RFC's
  client-driven framing (where validators never message one another). Making the mempool certified-only
  gives a single, uniform admission funnel on every node, and validator gossip keeps the committee
  well-connected without depending on a client to chase signatures.
- **Authoritative, non-rebuildable pin/certificate stores** kept out of the reindex loop, because an
  accidental wipe of the pin store is the highest-severity failure (it would permit equivocation).
- **Voiding the offending block** rather than crashing, so the ratification rule composes with the existing
  reorg machinery.

**Impact of not doing this:** Hathor keeps multi-second, probabilistic confirmation and remains
uncompetitive for point-of-sale and real-time payment rails — the exact market currently in focus.

# Prior art
[prior-art]: #prior-art

No system combines all three of this design's defining choices (consensusless per-UTXO certificate +
merged-mined-Bitcoin PoW hard tier + cert-as-block-validity-rule), but each tier has strong precedent:

- **FastPay** (Baudet, Danezis, Sonnino, AFT 2020, arXiv:2003.11506) — the direct ancestor: Byzantine
  consistent broadcast on `(account, sequence-number)`, `2f+1`/`3f+1` quorum, self-harm-only conflicts.
  Our per-UTXO pin is the UTXO analogue of its per-`(account, seq)` pin. Single-tier (no PoW).
- **Sui Lutris / Mysticeti** (arXiv:2310.18042, 2310.14821) — productionises the idea on an object model:
  owned objects take a consensusless certificate path, shared objects fall back to DAG-BFT, ~390 ms on
  mainnet. Owned objects ≈ our pinned UTXOs. Sui forgives equivocated objects at epoch boundaries — a
  cleaner stuck-object recovery than relying only on the slow tier (see Future possibilities).
- **Dash InstantSend + ChainLocks** — mechanically our Tier 1, in production since 2019: LLMQ masternodes
  threshold-sign each transaction input and skip conflicting inputs (DIP-0010/0006) — exactly our per-UTXO
  pin — aggregating into a single BLS `isdlock`. Key divergence: Dash's anti-reorg tier is another
  masternode quorum (ChainLocks), not objective merged-mined PoW, and the lock is not a block-validity
  rule.
- **Thunderella** (Pass & Shi, EuroCrypt 2018) — the academic template for an optimistic fast committee
  over a slow PoW fallback that ratifies and never loses safety. It is leader-driven and needs `>¾`
  honest; we chose leaderless, `>⅔`, and accountable.
- **Babylon** (arXiv:2408.01896) — fast BFT finality providers over Bitcoin with EOTS-based slashing;
  closest to our "fast accountable layer over a Bitcoin-grade base" thesis, but total-order BFT anchored
  via checkpoints rather than merged-mining with a cert-as-block-rule.
- **Casper FFG / Gasper (Ethereum)** and **GRANDPA (Polkadot)** — supply the `⅓`-accountable-safety math
  we reuse (two conflicting finalized decisions ⇒ `≥⅓` weight provably culpable), instantiated here
  *per UTXO*.
- **Syscoin Z-DAG** — our exact base (merged-mined Bitcoin + UTXO + ~60 s settlement) but a *probabilistic*
  fast tier with no certificate; it demonstrates the base we want paired with the fast tier we reject. The
  FastPay-style certificate is precisely what closes that gap.

**Lesson summary.** The fast tier is not speculative — Dash and Sui run it in production at ~0.4–1 s, and
the accountable-safety math is standard. Hathor's contribution is binding a *consensusless* fast
certificate to *merged-mined Bitcoin PoW* via a block-validity rule, so objective hashpower ratifies every
instant payment.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

To resolve through the RFC process / before stabilisation:

- **PoA vs PoS for the committee, and who slashes.** v1 ships a fixed, network-configured PoA committee.
  Moving to staking and economic slashing — turning equivocation evidence into a punishable offense — is
  deferred and determines whether soft finality is economically or only reputationally accountable.
- **Committee membership proof at handshake.** v1's overlay is "peers advertising `CAPABILITY_FINALITY`";
  safety does not depend on it (BLS verification is the real gate), but a handshake proof of committee
  membership would harden against DoS and improve privacy.
- **Stuck-UTXO recovery policy.** v1 leaves a wedged UTXO frozen until PoW settlement. A PoW tie-break
  ordering rule and/or Sui-style epoch-boundary unlock are open.
- **Multi-owner griefing.** A mechanism to stop a co-signer from wedging a joint transaction (input-level
  timeouts, refundable locks, or routing multi-owner spends to the total-order path).
- **Late-FC reconciliation edge cases.** The certified-only mempool covers the common case (a node holding
  a certified tx already has its FC). The residual case — a block references a certified tx the node has
  not seen — needs an explicit fetch path.

Out of scope for this RFC: the Nano-Contract total-order BFT design, validator fee/incentive and anti-spam
economics, and confidential-transaction interaction.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Batch / parallel certificate verification.** `blst`'s randomized multi-aggregate verification was not
  yet adopted; it would further raise verify throughput when a node validates many votes or certificates
  at once. Verification is embarrassingly parallel (no shared state), so it also scales across cores.
- **Binding FC-root block header (deferred PR G).** A 32-byte merkle root of the certified transactions in
  a block, committed in the header. v1 derives its binding security entirely from the consensus
  ratification rule (which needs the FC known only locally); a *binding* root would enable light-client
  header-only verification of soft finality — attractive for mobile wallets and external verifiers.
- **PoS staking and economic slashing.** Turn the two-contradictory-signatures equivocation evidence into
  on-chain fraud proofs and stake slashing.
- **Epoch governance, rotation, and epoch-boundary unlock** (from Sui Lutris) — forgive and unlock
  equivocated UTXOs at committee rotation, improving liveness without weakening safety.
- **ChainLocks-style fallback hard tier** (from Dash) — a validator-quorum anti-reorg backstop, useful
  transitionally if merged-mining hashrate on the fast tier is ever thin.
- **Nano-Contract total-order path.** The BFT counterpart for shared-state transactions, sharing the
  committee and the ratification rule with this fast path.
- **Confidential and cross-chain settlement.** The pin is over UTXO identity, not value, so the fast path
  is compatible in principle with Pedersen-committed confidential outputs; and the two tiers map naturally
  onto cross-chain transfers (softly final on certification, hard final on mainnet settlement).
