### Architecture Solution: Privacy-Preserving Private Access to Cloud-Deployed Frontier AI

**Part 1: Layer 1 (Authorization) and Layer 2 (Network Unlinkability)**

#### 1. Solution Overview

This architecture resolves the tension between Cloud Provider Economics and Individual Anonymity by cryptographically decoupling the user's billing identity from their inference requests. It achieves this without relying on blockchain consensus, opting instead for a high-throughput cryptographic protocol between a local client daemon, a blind Authorization Escrow ($E$), and the Cloud AI Provider ($CAI$).

**Relationship to IETF Privacy Pass (RFC 9576–9578):**
The ZK-Ticket Protocol extends the Privacy Pass paradigm with two additions that Privacy Pass's per-origin token buckets cannot support:
1. **Index-based quota management** — a client-side spending counter with ZK range proofs, enabling per-credit granularity without server-side balance tracking.
2. **Indexed nullifiers** — deterministic double-spend prevention tied to the spending counter.

Where Privacy Pass suffices (e.g., basic access gating without metered billing), implementers should prefer it.

**Glossary:**

| Term | Definition |
|---|---|
| **$U$ (User Daemon)** | Local background service managing secrets, proofs, and ticket buffers |
| **$E$ (Authorization Escrow)** | Independent third-party service that verifies ZKPs and mints Session Tickets |
| **$CAI$ (Cloud AI Provider)** | Entity hosting frontier AI models and enforcing the API gateway |
| **MasterSecret** | User-generated 256-bit secret, never leaves the client device |
| **VC (Verifiable Credential)** | BBS+-signed credential containing a commitment to MasterSecret and tier metadata |
| **Index** | Client-side monotonic counter tracking consumed credits |
| **Nullifier** | `Hash(MasterSecret \|\| Index)` — deterministic, one-way, used for double-spend prevention |
| **Credit** | Atomic billing unit of the protocol; mapping to compute is defined by $CAI$ |
| **Session Ticket** | Short-lived, single-use token signed by $E$, redeemable at $CAI$ for compute |

#### 2. Layer 1: Anonymous Authorization (The ZK-Ticket Protocol)

Layer 1 ensures that $CAI$ is financially compensated and can enforce strict rate limits without ever knowing which specific user is consuming the compute.

**2.1 Core Entities**

* **User Daemon ($U$):** A lightweight background service on the user's local edge device that manages cryptographic secrets and proofs. **One daemon instance per VC** — multi-device usage requires separate VCs.
* **Authorization Escrow ($E$):** An independent third-party service, operationally and legally separate from $CAI$ (see §2.13). Verifies ZKPs and mints single-use Session Tickets. Maintains a spent-nullifier database but no user database, balances, or identity records. May be operated by a privacy-focused non-profit, commercial provider, or consortium.
* **Cloud AI Provider ($CAI$):** The entity hosting the frontier AI models and enforcing the API gateway.

**2.2 Cryptographic Primitives**

**Blind Signature Scheme:** **BBS+ signatures** (or comparable: PS signatures, Coconut). Selected for blind issuance, selective disclosure within ZKPs, and efficient ZKP circuit integration. Implementation maturity note: audited libraries combining BBS+ blind issuance with ZKP circuit integration are limited — evaluate and audit before production use.

**ZKP Proving System:** **Groth16** (fast verification, per-circuit trusted setup) or **PLONK** (universal setup, larger proofs). Proving latency by platform: laptop/desktop 1–5s, mobile may require delegation, homelabs sub-second. Implementers should benchmark against target hardware before committing.

**Nullifier Hash Function:** **Poseidon** (ZKP-optimized) or **SHA-256** (widely audited, more expensive in-circuit). MasterSecret must have at least **256 bits of entropy**.

**2.3 Key Distribution & Trust Bootstrap**

$E$ must verify proofs about $CAI$-signed credentials, and $CAI$ must verify $E$-signed Session Tickets.

**Key publication:** Both $CAI$ and $E$ publish versioned key sets at well-known HTTPS endpoints (e.g., `/.well-known/p2fi-keys`). Each entry: `{ version, public_key, valid_from, valid_until }`. $E$ pins $CAI$'s key during onboarding; $CAI$ pins $E$'s key likewise.

**Key rotation:** Versioned keys with overlapping validity windows (e.g., 48-hour grace period). VCs signed with old $CAI$ keys remain valid until VC expiry. $E$ accepts proofs against any $CAI$ key version with unexpired VCs in circulation.

**Auditing $E$'s minting integrity:** $CAI$ knows total credits sold; $E$ mints tickets; $CAI$ redeems them. A surplus of redeemed tickets over issued credits indicates fraudulent minting. This statistical audit detects systematic over-minting. If detected, $CAI$ revokes trust in $E$'s key.

**ZKP circuit versioning:** Circuits versioned alongside key sets. $U$ includes circuit version in proof submission; $E$ rejects unsupported versions with a descriptive error.

**Clock synchronization:** All entities synchronize to NTP. Protocol tolerates clock skew of up to **30 seconds**.

**2.4 Phase 1: Credential Issuance (Blind Issuance Protocol)**

1. The User Daemon generates a `MasterSecret` (256-bit random) locally.
2. The user pays $CAI$ via a standard billing channel and requests a credential for a specific tier.
3. The Daemon computes a cryptographic commitment to `MasterSecret`, blinds it, and sends the blinded commitment to $CAI$ with proof of payment.
4. $CAI$ signs the blinded commitment with its BBS+ private key (attesting to the tier and credit limit) and returns the blind signature.
5. The Daemon unblinds the signature → valid VC stored client-side. $CAI$ has never seen the commitment in unblinded form and cannot recognize it later.

**Credential structure:**
```
VerifiableCredential {
  Commitment:   Commit(MasterSecret)       // blinded during issuance
  CreditLimit:  <integer, e.g., 1000>      // number of CUs
  Expiry:       <timestamp, e.g., 30 days> // VC validity period
  KeyVersion:   <CAI key version used>
  Signature:    BBS+_Sign_CAI(Commitment || CreditLimit || Expiry || KeyVersion)
}
```

**Multiple VCs:** A user may hold multiple active VCs simultaneously. The daemon prefers the VC with the nearest expiry to minimize wasted credits. Each VC has its own independent Index counter and MasterSecret.

**Tier anonymity:** If $CAI$ offers many granular tiers, the tier becomes a quasi-identifier. Mitigation: $CAI$ should offer **3–5 coarse, standardized tiers**. The credit limit is only revealed inside the ZKP (selective disclosure), never to $CAI$ at redemption time.

**2.5 Compute Credit Definition**

A **credit** is the atomic billing unit of the protocol. Each **Index increment** = 1 credit. `CreditLimit=1000` → Indexes 0–999.

How credits map to actual compute (tokens, GPU-seconds, request count) is entirely determined by $CAI$ and published at its well-known endpoint. $E$ operates purely on credit counts and has no knowledge of $CAI$'s pricing model. The daemon uses $CAI$'s published pricing for cost estimation.

$CAI$ unilaterally determines the credit cost per request. This is a trust assumption, mitigated by market competition — identical to traditional metered APIs.

**2.6 Phase 2: Anonymous Ticket Generation (The ZKP Execution)**

1. The User Daemon increments its local index (e.g., `Index=42`).
2. The Daemon executes a local **ZKP circuit** proving to $E$:
   * *Authenticity:* "I hold a VC signed by $CAI$ (key version N)."
   * *Capacity:* "`Index < CreditLimit`" (ZK Range Proof).
   * *Freshness:* "`VC.Expiry > proof_timestamp`" — where `proof_timestamp` is a **public input**.
   * *Knowledge:* "I know the `MasterSecret` corresponding to the VC's commitment."
3. The daemon generates an **Indexed Nullifier**: `Nullifier = Hash(MasterSecret || Index)`.
4. The Daemon submits the ZKP, Nullifier, and `proof_timestamp` to $E$ **via the L2 proxy network** (see §3).

**Timestamp verification:** `proof_timestamp` is a public input readable by $E$. $E$ rejects timestamps more than 30 seconds from its own clock, preventing use of expired VCs. Privacy implication: $E$ sees the proof timestamp at ~30-second granularity — too coarse to degrade unlinkability in a sufficiently large user population.

**Batch proof generation:** Multiple `(ZKP, Nullifier)` pairs for sequential Indexes can be generated in parallel on multi-core hardware.

**2.7 Phase 3: Verification & Minting**

1. $E$ validates `proof_timestamp` against its clock (30-second window).
2. $E$ checks Nullifier(s) against its spent-nullifier database.
3. If unique, $E$ verifies each ZKP against $CAI$'s published public key.
4. $E$ records Nullifiers and mints corresponding Session Tickets.

**Batch submission:** Multiple `(ZKP, Nullifier)` pairs in a single request. Atomic validation — if any proof is invalid or nullifier is spent, the entire batch is rejected.

**Session Ticket structure:**
```
SessionTicket {
  Nonce:       <cryptographically random, unique>
  Quota:       1 credit
  Expiry:      <timestamp, e.g., 15 minutes from issuance>
  Signature:   Sign_E(Nonce || Quota || Expiry)
}
```

**Ticket binding:** Transmitted over TLS terminating at $CAI$ (or HPKE-encrypted in OHTTP mode). Relay nodes cannot observe or extract tickets. In multi-hop mode, onion encryption provides equivalent protection.

**Single-use enforcement:** $CAI$ verifies $E$'s signature, checks expiry (with 30-second tolerance), and records the Nonce in a spent-ticket set. Duplicate nonces rejected. Expired nonces garbage-collected.

**Ticket validity during inference:** A ticket presented before its Expiry remains valid for the duration of the inference request it funds, even if inference completes after Expiry.

**$E$ API (sketch):**

```
POST /mint

Request:  { "circuit_version": "v1", "proofs": [{ "zkp": ..., "nullifier": <hex>, "proof_timestamp": <unix>, "cai_key_version": <int> }, ...] }
Response: { "tickets": [{ "nonce": "...", "quota": 1, "expiry": "...", "signature": "..." }, ...] }
Error:    { "error": "<NULLIFIER_SPENT|PROOF_INVALID|TIMESTAMP_SKEW|CIRCUIT_UNSUPPORTED|RATE_LIMITED>", "message": "..." }
```

**2.8 Multi-Ticket Redemption at $CAI$**

A single inference request may consume multiple CUs. The daemon attaches a ticket bundle to the request.

**Validation and execution:** $CAI$ uses a **validate-all-then-execute** model:
1. Validate all submitted tickets atomically (signature, expiry, nonce uniqueness).
2. If **any** ticket is invalid, reject the entire request — no nonces recorded.
3. If all valid, record nonces as spent and begin inference.

**Shortfall (underestimation):** $CAI$ pauses inference, returns `HTTP 402` with `{ "required_cu": N, "consumed_cu": M, "shortfall": N - M }`. The daemon sends additional tickets; $CAI$ resumes. Session context preserved server-side for a configurable window (e.g., 60 seconds).

**Surplus (overestimation):** On completion, $CAI$ reports actual consumption. Surplus tickets' nonces are **not** recorded as spent — they remain reusable within their expiry window. The daemon returns unused tickets to the pre-fetch buffer.

**2.9 Dynamic Asynchronous Top-Ups**

**Pre-fetching:** The daemon maintains a buffer of 5–10 Session Tickets ahead of current consumption, refilled asynchronously.

**Cover traffic:** To prevent timing correlation between ticket-fetch patterns at $E$ and session activity at $CAI$, the daemon issues background ticket fetches at randomized intervals. Cover traffic uses real Indexes (consuming credits) — $E$ must not distinguish cover from real requests.

Cover traffic parameters (user-configurable):
* **Default:** ~5–10% additional credit overhead.
* **High-privacy:** More aggressive, higher cost.
* **Cost-sensitive:** Disabled; pre-fetch buffer alone provides temporal decoupling.

**2.10 Credential Revocation**

There is no mechanism for $CAI$ to revoke a specific VC after issuance — by design, as any such mechanism would require identifying the VC.

* **Short-lived VCs:** 30-day expiry. Periodic re-issuance bounds the damage window of a compromised MasterSecret.
* **User-initiated revocation:** Burn remaining nullifiers by submitting them to $E$, then re-purchase.
* **Session-level termination:** $CAI$ can terminate any session for policy violations without affecting future sessions.

**Key and state loss:** MasterSecret loss = irrecoverable credit loss. Index counter loss can be recovered via **binary search** — probe $E$ with candidate nullifiers over `[0, CreditLimit)`, requiring `O(log(CreditLimit))` probes (~10 for a 1000-credit VC). Privacy note: the probe burst is detectable by $E$; use as last resort. The daemon should write Index state to durable encrypted storage after every increment.

**2.11 Trust & Safety**

This architecture provides **per-session incognito** — $CAI$ cannot link sessions to a billing identity or across sessions. $CAI$ retains full visibility into plaintext content within each session (subject to Layer 3, covered separately).

**In-session safeguards:** $CAI$ applies standard content-safety mechanisms: model-level filtering, per-session behavioral analysis, and session termination for policy violations.

**Out of scope by design:** Identity-level bans and cross-session behavioral profiling — impossible by design to preserve the incognito guarantee.

**2.12 Temporal Rate Limiting**

The ZK-Ticket protocol enforces *total* quota (Index < CreditLimit) but not *temporal* rate limits. DDoS prevention:
* **Session Ticket expiry** (15 minutes) prevents indefinite stockpiling.
* **$CAI$ gateway rate limiting** per-ticket and per-relay-IP.
* **$E$ global minting rate limits** applied uniformly.

**2.13 Trust Model & Collusion Resistance**

**Independence requirements:** $E$ must be legally and operationally independent from $CAI$, ideally in separate jurisdictions. $E$'s architecture must minimize logging — ideally only the spent-nullifier set, no request metadata.

**$E$'s observability:** Aggregate metrics only (proofs/minute, nullifier DB size, error rates) — no per-request logging.

**What collusion requires:** Correlating ticket-minting timestamps with session-start timestamps. Pre-fetching, cover traffic, and L2 proxying degrade this correlation.

**Residual risk:** In small user populations (< 100 active users) with identifiable tier configurations, statistical deanonymization remains feasible. Anonymity set size is a fundamental constraint.

**$E$ compromise:** Can mint fraudulent tickets (detected by aggregate audit), log metadata for correlation (mitigated by minimal-logging architecture), or selectively deny service. Compromise does not expose plaintext prompts (only $CAI$ sees those) or network identity (only relays see that).

**Future — Threshold $E$:** Distribute across $k$-of-$n$ independent operators using threshold cryptography (e.g., Coconut). No single operator can deanonymize; provides fault tolerance. Not required for initial design.

**Compromised User Daemon:** Out of scope — no network protocol can protect against a compromised endpoint. The protocol assumes a trusted client environment.

**Sybil resistance** is purely economic — equivalent to traditional API access.

**2.14 Nullifier Database Management**

**Growth model:** 32 bytes per nullifier. At 1M users × 1000 credits/month ≈ 32 GB/month.

**Garbage collection:** Nullifiers expire with their VC (e.g., 30-day rolling window).

**Storage:** Bloom filters are **not** recommended — false positives cause irrecoverable credit rejection.

#### 3. Layer 2: Network Unlinkability (Proxy Routing)

Network metadata (IP addresses, packet timing) can deanonymize users. L2 severs this link for **all protocol traffic** — both to $CAI$ and to $E$.

**3.1 Operational Modes**

**Mode A: OHTTP (RFC 9458)** — Single-relay, low-latency (~10–50ms overhead).
* Daemon encrypts payload under destination's HPKE key, sends to OHTTP Relay.
* Relay strips IP headers, forwards opaque blob. Cannot read payload; destination cannot see client IP.
* **Requires:** Relay and destination do not collude.

**Mode B: Multi-Hop Onion Routing** — 2–3 relays, higher latency (50–200ms/hop), stronger anonymity.
* Layered encryption: each relay peels one layer, knows only previous and next hop. Innermost layer encrypted under destination's key.
* **Requires:** At least one honest relay in the circuit.
* Existing networks (Tor, Nym) may serve as Mode B implementations; evaluate latency for interactive inference.

Mode A is the default. Mode B when no single relay operator can be trusted.

**3.2 Relay Diversity & Federation**

* **Minimum:** 3 independent relay operators in different jurisdictions.
* In Mode B, select relays from **different operators** per hop.
* Relay selection varies across sessions, stable within a session.

**Relay incentives:** Sponsorship, bundling with existing privacy infrastructure (VPN/CDN operators), or direct payment via a **separate billing mechanism** (not Session Tickets — that creates a circular dependency since reaching $E$ requires the relay). Alternatively, relays treat $E$-bound traffic as unpaid infrastructure and charge only for $CAI$-bound inference traffic.

**3.3 Traffic Analysis Mitigations**

* **Request padding:** Fixed size buckets (1KB, 4KB, 16KB, 64KB).
* **Timing jitter:** 10–100ms random delays on outbound requests.
* **Cover traffic:** Background ticket fetches (§2.9).
* **Relay mixing:** Large user population provides natural anonymity set. Relay operators should publish throughput metrics.

**Residual risk:** A global passive adversary can correlate flows even with these mitigations. Full resistance requires high-latency mix networks, incompatible with interactive inference. The architecture prioritizes usable latency.

#### 4. Protocol Flow Summary

```
Phase 1: Credential Issuance (one-time, identity-linked)

  U                              CAI
  |--- payment + blinded ---------->|
  |    commitment                   |
  |<-- blind BBS+ signature --------|
  U unblinds → stores VC locally

Phase 2–3: Ticket Generation & Minting (per-credit, anonymous)

  U                    Relay            E
  | generate ZKP(s)                     |
  |--- [HPKE/onion] --->|               |
  |    batch: [{ZKP,    |--- forward -->|
  |    Nullifier,       |               | validate, verify, mint
  |    timestamp}, ...] |<-- tickets ---|
  |<--- [encrypted] ----|               |
  U stores tickets in pre-fetch buffer

Phase 4: Inference (per-session, anonymous)

  U                    Relay           CAI
  |--- [HPKE/onion] --->|               |
  |    prompt + tickets |--- forward -->|
  |                     |               | validate tickets, run inference
  |                     |               |
  |                     |               | if shortfall:
  |                     |<-- 402 -------|
  |<--- [encrypted] ----|               |
  | fetch more tickets (Phase 2–3)      |
  |--- [HPKE/onion] --->|--- forward -->| resume
  |                     |<-- response --|
  |<--- [encrypted] ----|               |
  | return unused tickets to buffer     |
```

#### 5. Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **$E$ unavailable** | No new tickets; pre-fetch buffer provides ~15 min runway | HA deployment; daemon surfaces clear error with remaining ticket count |
| **Nullifier DB corruption** | Double-spend risk; financial loss to $CAI$ | Replication + WAL; $E$ publishes nullifier count/Merkle root for $CAI$ audit (daily minimum) |
| **Daemon state loss** | MasterSecret loss = irrecoverable credits; Index loss = recoverable via binary search | Encrypted backups; durable write-after-increment |
| **Clock skew** | False ticket rejection or security degradation | 30-second tolerance; NTP required |
| **$CAI$ key rotation** | VCs signed with old keys must stay verifiable | Overlapping validity windows; $E$ accepts all unexpired key versions |
| **Relay network failure** | Loss of L2 privacy | Daemon must **not** silently fall back — explicit user consent required |
| **Concurrent daemons** | Nullifier collisions, state corruption | One daemon per VC (enforced via lock file) |

#### 6. Security Properties & Limitations

**Properties achieved** (contingent on correct cryptographic instantiation):
* **Unlinkability:** $CAI$ cannot link sessions to billing or network identity.
* **Unprofilability:** Sessions are cryptographically unlinkable across time.
* **Double-spend resistance:** $E$'s nullifier set prevents credit reuse.
* **Abuse resistance:** In-session safeguards, ticket expiry, economic Sybil resistance.

**Threat model hierarchy** (by likelihood):
1. **Compromised user device** — out of scope; mitigated by endpoint security.
2. **E + CAI collusion** — mitigated by jurisdictional separation, pre-fetching, cover traffic, L2. Degraded in small populations.
3. **Traffic analysis by relays** — mitigated by relay diversity, padding, jitter. Mode B provides stronger resistance.
4. **$E$ compromise** — detected by aggregate audit. Damage limited to metadata correlation.
5. **Global passive adversary** — not fully defended; inherent latency/anonymity tradeoff.

**Additional limitations:** Credit-to-compute billing is trust-based (market-mitigated). Key loss is irrecoverable. Identity-level bans are impossible by design. Formal security proofs are deferred to implementation phase.
