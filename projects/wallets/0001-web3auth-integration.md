- Feature Name: web3auth_social_login
- Start Date: 2026-03-15 (updated 2026-04-14)
- RFC PR: HathorNetwork/rfcs#106
- Hathor Issue: HathorNetwork/internal-issues#516
- Author: André Abadesso <andre@hathor.network>

# Summary
[summary]: #summary

Integrate Web3Auth (MetaMask Embedded Wallets) into Hathor wallets so users can onboard via social login (Google, Apple, email) instead of managing a 24-word seed phrase. This RFC is the **foundation document**: what Web3Auth is, how we talk to it, and the operational steps required at a company level to ship the integration. The per-layer implementation designs live in companion RFCs:

- **Wallet-lib + mobile UI**: [internal-rfcs#46](https://github.com/HathorNetwork/internal-rfcs/pull/46)
- **Wallet-service support**: [internal-rfcs#47](https://github.com/HathorNetwork/internal-rfcs/pull/47)
- **Library PoC**: [hathor-wallet-lib#1062](https://github.com/HathorNetwork/hathor-wallet-lib/pull/1062)

# Motivation
[motivation]: #motivation

Seed-phrase onboarding is the single biggest drop-off point for non-crypto-native users. Web3Auth replaces it with social login while keeping the wallet non-custodial — the full private key is never held by Web3Auth, Hathor, or any single party.

Expected outcomes:
- Sign in with Google / Apple / email in seconds, no seed backup.
- Lower fund-loss rate (key recovery is tied to social identity + an explicit recovery share, not a piece of paper).
- Parallel onboarding path — existing HD-wallet users are unaffected.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## What Web3Auth is

Web3Auth (owned by Consensys / MetaMask, formerly Torus Labs) is a key-management service that issues **one secp256k1 private key per authenticated user**. The key is protected by a 2-of-3 Shamir Secret Sharing scheme:

- **ShareA** — "social share", distributed across a 5-node Auth Network operated by Torus Labs. Released after OAuth authentication with the user's social provider.
- **ShareB** — "device share", stored locally on the user's device (Keychain on iOS, Keystore on Android, `safeStorage` on Electron, IndexedDB on web).
- **ShareC** — "recovery share", controlled by the user (downloadable backup file, security questions, or secondary device).

Any two shares reconstruct the private key client-side. The key is never transmitted and never stored whole on any server. If Web3Auth disappears, users with ShareB + ShareC still have their key.

The SDK exposes the reconstructed key via `provider.request({ method: 'private_key' })` as a raw 32-byte hex string. That is all we get from Web3Auth — **no BIP32 chain code, no HD tree, no xpub**.

## Key architectural decision: single-key wallet, NOT BIP32-from-raw-key

Because Web3Auth returns a raw private key with no chain code, the natural temptation is to use it as BIP32 master entropy and derive a full HD tree. **We explicitly do not do this.** The decision and its rationale live in [internal-rfcs#46](https://github.com/HathorNetwork/internal-rfcs/pull/46); summarized:

- Fabricating a chain code from the raw key is a non-standard cryptographic construction.
- It breaks cross-app portability — the same Web3Auth login would produce different Hathor addresses depending on each app's chosen chain-code construction, defeating the point of using a portable auth provider.
- It doesn't simplify signing (the only signable key is still the one raw key) and doesn't fix wallet-service compatibility.

Instead, the wallet-lib gains a **single-key wallet mode**: one raw private key, one address, signing routed through the existing `setExternalTxSigningMethod` hook. Web3Auth-issued wallets are single-key wallets.

## Trust model in one table

| You trust | With | They cannot |
|---|---|---|
| **Torus Labs** (operates all 5 Auth Network nodes) | Honest node operation; availability of ShareA sub-shares | Reconstruct the full key (they hold only ShareA; ShareB and ShareC live elsewhere) |
| **OAuth provider** (Google/Apple/etc.) | Authenticating the user honestly | Access any key material |
| **User's device** | Not being compromised by malware | — (compromise leaks ShareB) |
| **Web3Auth SDK supply chain** (npm / CDN) | Serving unmodified JS | — (compromised SDK could exfiltrate the reconstructed key at runtime) |

The single material risk specific to Web3Auth's architecture is that **all 5 nodes are operated by Torus Labs**. A privileged insider could modify node software to log released sub-shares — they'd get ShareA but would still need ShareB or ShareC to reconstruct a key. Web3Auth has SOC 2 Type II and the node software is open source ([`torusresearch/torus-node`](https://github.com/torusresearch/torus-node)); we accept this risk because every alternative has worse properties (see "Alternatives").

## Recovery scenarios

| Situation | Outcome |
|---|---|
| Normal login on same device | ShareA (from nodes) + ShareB (from device) → key |
| New device (old device lost) | ShareA (from nodes) + ShareC (recovery backup) → key |
| Web3Auth Auth Network down | ShareB (device) + ShareC (recovery) → key |
| Web3Auth down AND no recovery share set up | **Funds are lost.** |

The last row is why enforcing recovery-share setup during onboarding is a hard requirement (see "Open decisions").

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Integration architecture

```
User → OAuth provider → Web3Auth SDK (client-side)
                            ↓
                        raw secp256k1 private key
                            ↓
           ┌─ derive public key + Hathor address locally
           ↓
    HathorWallet({ privateKey, publicKey, preCalculatedAddresses: [address] })
           ↓
    wallet.setExternalTxSigningMethod(web3AuthSigner)
           ↓
    wallet starts; UI shows one address, uses existing tx flow
```

No BIP32 derivation. No seed. No xpub. Everything downstream of the raw key is handled by the single-key wallet mode defined in internal-rfcs#46 and implemented in hathor-wallet-lib#1062.

## SDK choice

**Use the Plug-and-Play (PnP) SDK family**, not Core Kit:

| | PnP | Core Kit |
|---|---|---|
| Pre-built OAuth UI | yes | no |
| Share management handled for us | yes | we handle it |
| Time to ship | weeks | months |
| UI customization | limited | full |
| MPC-TSS available | yes (Enterprise) | yes (Enterprise) |

Per-platform SDKs:
- **Mobile** (phase 1 target): `@web3auth/react-native-sdk` — integration details in internal-rfcs#46.
- **Desktop** (future): `@web3auth/modal` (web build running in Electron). Electron's `safeStorage` API for the device share instead of localStorage.
- **Web** (future): `@web3auth/modal` again, device share in IndexedDB.

We start with **SSS (non-MPC)** pricing tier. MPC-TSS (where the private key is never reconstructed in memory) is Enterprise-only and is a future upgrade; SSS is sufficient for launch.

## Operational setup — what we actually have to do

This is the list of company-level actions required **before any integration code runs**. These are tracked as explicit setup steps, not as implementation work:

### 1. Create the Web3Auth account and project

- Sign up at [`dashboard.web3auth.io`](https://dashboard.web3auth.io).
- **Account owner**: a single named person or role account at Hathor (billing + admin). Recommend a role account (e.g. `ops@hathor.network`) so access survives team changes.
- Create **two projects**:
  - `hathor-wallet-dev` on the Sapphire **Devnet** (free, used by engineering and QA).
  - `hathor-wallet-prod` on the Sapphire **Mainnet** (paid tier, used by shipped apps).
- Each project yields a distinct `clientId`, baked into app builds via env vars.
- **Critical**: devnet and mainnet are **separate key universes**. The same Google login produces **different private keys** on devnet vs mainnet. Users onboarded on a staging build cannot recover on prod (and vice versa). Document this in the QA runbook.

### 2. Pick a pricing tier

As of writing:

| Tier | Monthly active wallets | Approx. cost | Includes |
|---|---|---|---|
| Free | 1,000 | — | SSS only |
| Growth | 3,000 | $69/mo | SSS |
| Scale | 10,000 | $399/mo | SSS, fiat on-ramp |
| Enterprise | custom | custom | MPC-TSS, custom SLA |

**Recommendation**: start on **Growth** for launch headroom; upgrade to Scale if uptake exceeds 2k wallets; revisit Enterprise (MPC-TSS) only if security review requires it or we exceed Scale limits. Track MAU to forecast tier transitions.

### 3. Configure OAuth providers

Each social provider has to be (a) configured in the Web3Auth dashboard and (b) registered as an OAuth app with the provider itself.

| Provider | What we need | Who owns the credentials |
|---|---|---|
| **Google** | Google Cloud project + OAuth 2.0 clients (iOS + Android + web variants) | Hathor ops |
| **Apple** | Apple Developer account + Sign in with Apple service ID + key | Hathor ops (required on iOS per App Store guideline 4.8) |
| **Email-passwordless** | Just enable in dashboard; Web3Auth handles delivery | — |
| **Discord, Twitter, etc.** | Per-provider OAuth app registration | Hathor ops, when enabled |

**Phase 1 minimum**: Google + Apple + email-passwordless. Anything else is deferred post-launch.

Redirect URIs / allowed bundle IDs must be configured per platform:
- iOS: URL scheme + Apple Team ID + bundle identifier.
- Android: package name + SHA-1 and SHA-256 signing certificate fingerprints (one set per build variant: debug / staging / production).
- Web/desktop: allowed origins and redirect URLs.

### 4. Verifier strategy

Web3Auth ties keys to a `(verifier, verifier_id)` tuple. Two options:

- **Shared verifiers** (Web3Auth's default `tkey-google`, `tkey-apple`, etc.). Fast to set up. Users who sign into multiple apps that use the same shared verifier get the **same private key** across those apps. This is either a feature (cross-app portability) or a leak (other apps can derive our users' keys) depending on what you're optimizing for.
- **Custom verifiers**. We run our own JWT signing infrastructure; Web3Auth only validates JWTs we sign. Isolated key space, no cross-app leakage. Higher ops cost.

**Recommendation**: **custom verifiers for Phase 1**, keyed on Hathor-signed JWTs. The cross-app-portability benefit of shared verifiers is not worth the security posture of "anyone running a Web3Auth app could derive our users' Hathor keys." This adds a one-time backend build (JWT signing service).

### 5. Recovery-share policy

Open decisions (see below) need resolution here. Options:

- **Mandatory**: onboarding blocks until the user sets up a recovery share.
- **Strongly encouraged**: offered during onboarding, dismissible with a clear warning.
- **Optional / later**: set up from Settings after onboarding.

**Recommendation**: **mandatory**. The "Web3Auth down + no recovery share = funds lost" scenario is severe enough that the UX friction of requiring a recovery share is the lesser evil. Product decision, not engineering.

### 6. Session and key-lifetime policy

- **Web3Auth session TTL** — accept default (~24h) unless legal requires otherwise.
- **App-level PIN lock** — encrypt the reconstructed private key with a user-chosen PIN; decrypt on unlock. Matches the existing HD-wallet seed encryption model on mobile.
- **Device share refresh** — Web3Auth rotates device shares on a configurable cadence; accept default.

### 7. Monitoring and incident response

- Subscribe to Web3Auth's status page; feed alerts into our on-call.
- Instrument our app to log and alert on Web3Auth SDK errors and OAuth failures (without logging PII).
- Document a support playbook: what does mobile support tell a user when the Auth Network is degraded? (Already-onboarded users can still unlock with their PIN; new onboarding is blocked until service restores.)

### 8. Legal / compliance

- Privacy policy updated to disclose Web3Auth as a sub-processor.
- DPA (Data Processing Agreement) with Web3Auth — request during project setup.
- Terms of service updated to disclose the trust model (Torus Labs operates the nodes, recovery-share responsibility, etc.).

## Existing wallet compatibility

The integration is **additive**. Existing HD wallets (seed-phrase, hardware) are unchanged. The user-type decision happens at onboarding:

```
Onboarding
 ├─ "I have a seed phrase" → existing HD flow
 ├─ "Create new wallet" → existing HD flow
 └─ "Sign in with Google / Apple / email" → Web3Auth flow (this RFC)
```

Once started, a Web3Auth wallet and an HD wallet produce identical downstream behavior (send, receive, tokens, nano contracts) because the wallet-lib abstracts over both via the single-key vs HD modes.

# Drawbacks
[drawbacks]: #drawbacks

- **Third-party dependency**: Web3Auth is now on our critical path. Their Auth Network going dark degrades onboarding; only users with recovery shares can move funds on a new device.
- **Centralized node operation**: all 5 nodes are Torus-Labs-operated. Mitigated, not eliminated, by multi-share architecture + open-source node code.
- **SSS tier reconstructs key in memory**: vulnerable to browser extensions or device malware during the signing window. MPC-TSS avoids this but is Enterprise-only.
- **Supply-chain exposure**: the Web3Auth JS SDK runs in our users' app context. A compromised SDK could exfiltrate the reconstructed key at runtime. Mitigate via dependency pinning, Subresource Integrity (where applicable), and periodic SDK audits.
- **User mental model**: "Sign in with Google" feels custodial; users may assume Google can recover their funds. Clear onboarding copy required.
- **Maintenance surface**: one more onboarding path, one more SDK, one more set of OAuth integrations to maintain.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why Web3Auth

- **Non-custodial** by construction (no single party holds the full key).
- **secp256k1 native** — no curve translation for Hathor's key model.
- **Battle-tested**: powers MetaMask Embedded Wallets; 20M+ MAU across the platform.
- **Works offline of Web3Auth**: ShareB + ShareC is sufficient.

## Alternatives rejected

- **Magic (magic.link)** — uses AWS KMS; effectively custodial. Incompatible with our non-custodial stance.
- **Privy** — strong EVM focus, weak non-EVM support.
- **Dynamic** — primarily EVM, MPC roadmap not yet shipped.
- **Custom MPC (tss-lib)** — extreme engineering and audit cost for a non-differentiating capability.

## Why single-key mode, not HD-from-raw-key

Covered in internal-rfcs#46's "Rationale and alternatives" section. Summary: fabricating a chain code for BIP32 breaks cross-app portability (a selling point of Web3Auth), is a non-standard crypto construction, and doesn't simplify signing or fix wallet-service compatibility.

# Open decisions
[open-decisions]: #open-decisions

These need answers before Phase 1 ships. None are research questions — they are choices that require a named owner:

- [ ] **Recovery-share enforcement policy**: mandatory / strongly encouraged / optional? (Product owns this.)
- [ ] **Custom vs shared verifiers**: RFC recommends custom; operational cost of running JWT signing infra. (Security + ops sign-off.)
- [ ] **Pricing tier at launch**: Growth vs Scale vs starting Free and upgrading. (Finance sign-off.)
- [ ] **OAuth provider set at launch**: confirm Google + Apple + email is the Phase 1 minimum, nothing more. (Product.)
- [ ] **Web3Auth account owner at Hathor**: single named human or role account. (Ops.)
- [ ] **MPC-TSS requirement**: security review — is SSS acceptable for launch, or does MPC need to gate release? (Security.)

# Prior art
[prior-art]: #prior-art

- **MetaMask Embedded Wallets** — MetaMask's production deployment of Web3Auth. The reference integration.
- **Binance Web3 Wallet** — similar 3-share model (cloud / device / recovery password). Proprietary but validates the market.
- **Argent** — social recovery on Ethereum via guardian contracts. Different mechanism, same UX goal.
- **Phantom (Solana)** — explored seedless flows; kept seed phrases as primary. Cautionary tale about UX complexity.

Lesson across all: users prefer social login, but the security model must be communicated clearly or they develop false expectations about recoverability.

# Related work
[related-work]: #related-work

This RFC is the foundation for the Web3Auth initiative. The per-layer implementation work is tracked separately:

- **[internal-rfcs#46](https://github.com/HathorNetwork/internal-rfcs/pull/46)** — Wallet-lib single-key mode + mobile onboarding UI. Answers "how does the library accept a raw private key and route signing through the external-signer hook," and "what does the mobile onboarding / lock / settings UI look like."
- **[internal-rfcs#47](https://github.com/HathorNetwork/internal-rfcs/pull/47)** — Wallet-service support for single-key wallets. Answers "how do we identify and authenticate a wallet that has no xpub." Stacked on #46.
- **[hathor-wallet-lib#1062](https://github.com/HathorNetwork/hathor-wallet-lib/pull/1062)** — Library PoC with unit and integration tests proving the single-key mode works end-to-end.

# Future possibilities
[future-possibilities]: #future-possibilities

- **MPC-TSS upgrade** (Enterprise tier): private key never reconstructed in memory; eliminates the memory-exposure class of attacks. Gate on MAU + security review.
- **Passkey / WebAuthn** as an authentication factor in addition to or instead of social login. Supported by Web3Auth.
- **Desktop wallet integration**: once mobile ships and is validated, the same architecture extends to Electron with `safeStorage` for the device share.
- **Pre-generated wallets** (Scale tier): enables airdrop or marketing flows where users receive tokens before creating an account.
- **Fiat on-ramp** (Scale tier): completes the "zero-to-crypto" path for non-crypto-native users.
