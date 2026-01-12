# Bitcoin Pre-Contracts (BPC)

**A Deterministic Authorization Framework for Bitcoin Settlement**

* **Specification Version:** 1.0.0
* **Status:** Public Beta - Feedback Welcome
* **Date:** 2026
* **Author:** rosiea
* **Contact:** [PQRosie@proton.me](mailto:PQRosie@proton.me)
* **Licence:** Apache License 2.0 — Copyright 2026 rosiea

---

## 0. Introduction

### 0.1 What BPC Does

Bitcoin Pre-Contracts (BPC) enables **deterministic, auditable authorization** before Bitcoin transactions are constructed.

BPC solves a fundamental problem: how to enforce complex preconditions—multi-party approvals, time constraints, verifiable proofs—before any broadcast-valid transaction exists, while preserving Bitcoin's sovereignty model and requiring no consensus changes.

**Core capabilities:**

- **Refusal-first enforcement** — Wallets refuse to construct transactions until all preconditions are satisfied
- **Deterministic evaluation** — Identical inputs always produce identical authorization decisions across implementations
- **Verifiable time authority** — All timing decisions derive from cryptographically signed, Bitcoin-anchored time
- **Template binding** — Authorization commits to specific transaction structure, preventing post-approval mutation
- **Standard Bitcoin settlement** — Final transactions are normal Bitcoin L1 transactions

**The insight:** No broadcast-valid spend artifact exists until preconditions are satisfied.

### 0.2 Philosophy

BPC is built on a small set of Bitcoin-native principles that prioritise
sovereignty, determinism, and explicit authority boundaries.

#### 0.2.1 Sovereignty Preservation

BPC preserves the core Bitcoin principle that **the signer is sovereign**.

No component in a BPC system is permitted to compel signing, infer consent,
or proceed on behalf of a key holder. Wallets and signers retain final control
over whether a transaction is constructed and signed.

BPC does not automate signing. It defines **deterministic refusal rules**
that wallets may enforce prior to signing.

#### 0.2.2 Bitcoin as Root Anchor

All authority in BPC ultimately derives from Bitcoin itself.

- Time authority is anchored to a Bitcoin-inscribed profile (Epoch Clock)
- Settlement occurs exclusively via standard Bitcoin L1 transactions
- No changes are made to Bitcoin consensus, Script, or transaction validity rules

BPC operates entirely *before* transaction construction. Once a transaction
is broadcast, BPC has no further role.

#### 0.2.3 Decentralisation Without New Consensus

BPC introduces no new ledger, token, or consensus mechanism.

- Mirrors provide redundancy, not authority
- Evaluators verify evidence; they do not assert truth
- Verification is local and deterministic
- No single service is trusted by default

This preserves Bitcoin’s decentralised trust model while enabling richer
pre-transaction coordination.

#### 0.2.4 Refusal-Driven Design (Informative)

BPC follows **Refusal-Driven Design (RDD)**.

In Refusal-Driven Design, systems are constructed such that:
- the default and safe behaviour is refusal,
- progress occurs only when required inputs are present and verifiable,
- no executable or spend-authorising artefact exists prior to authorisation.

In BPC, refusal is not an error condition or fallback path. It is the
**primary enforcement mechanism**.

Specifically:
- absence of required evidence results in refusal (fail closed),
- authorisation produces no broadcast-valid spend artefacts before `ALLOW`,
- canonical binding prevents post-authorisation mutation,
- determinism is enforced through canonical encoding and explicit authority sources.

RDD is a design doctrine that applies across the PQ ecosystem.
Normative enforcement rules are defined elsewhere in this specification
(e.g. Security Invariants, Execution Gate).

---

This philosophy ensures that BPC systems degrade safely, remain auditable,
and preserve signer sovereignty under all failure modes.


### 0.3 Positioning

BPC operates **before** transaction construction, not during or after.

It is:
- A **refusal framework** for authorization discipline
- A **precondition evaluator** for complex spending policies
- A **binding mechanism** between approval and execution

It occupies the space: **deterministic off-chain evaluation + standard Bitcoin settlement**.

### 0.4 Cryptographic Scope (Informative)

BPC v1.0.0 is specified as a **Minimum Viable Protocol (MVP)** with respect to cryptographic primitives.

This specification defines:
- Deterministic structure and canonical encoding
- Refusal-first enforcement semantics
- Explicit authority and binding boundaries
- Extension points for future cryptographic suites

BPC's security value derives from **authorization discipline and execution gating**, not from replacing Bitcoin's consensus cryptography. This ensures immediate deployability on current Bitcoin infrastructure while avoiding premature cryptographic standard lock-in.

### 0.5 Quantum Awareness (Informative)

BPC is **quantum-aware** in architecture but **classically settled** on Bitcoin L1.

| Component | Status | Quantum Property |
|---------|--------|------------------|
| Epoch Clock signatures | Active | Post-quantum (ML-DSA-65) |
| PreContractOutcome signatures | Deployment-defined | MAY be post-quantum via `sig_alg` |
| Approvals signatures | Deployment-defined | MAY be post-quantum via `sig_alg` |
| Bitcoin settlement (L1) | Fixed | Classical (secp256k1) |

The architecture does not assume perpetual secrecy of classical keys. Authorization is not reduced to a single cryptographic primitive. Time, structure, and execution gating remain meaningful even if some primitives weaken.

Post-quantum guarantees in BPC apply to the **authorization layer** only. Full post-quantum settlement requires Bitcoin consensus changes and remains out of scope.

---

## 1. Purpose and Scope

### 1.1 Core Purpose

BPC defines a deterministic architecture for enforcing **contractual preconditions** before a Bitcoin transaction can be constructed and broadcast.

BPC ensures:
- A wallet or engine **refuses** (fails closed) on missing, stale, ambiguous, or unverifiable evidence
- Evaluation is deterministic: identical inputs produce identical outputs
- Bitcoin remains unchanged: settlement is normal L1 execution and finality
- An audit trail exists for every evaluation decision

### 1.2 Use Cases

BPC excels in scenarios requiring enforceable preconditions:

**Delivery-vs-Payment (DvP)**  
Neither party sends first. Conditions gate transaction construction. Both parties provide evidence; transaction only exists after mutual verification.

**Multi-Party Treasury**  
Require N-of-M approvals before any broadcast-valid transaction exists. Policy enforcement happens before signing, not after.

**Time-Bounded Releases**  
Funds cannot be constructed into a spendable transaction until verified time conditions pass. No reliance on block timestamps or system clocks.

**Proof-Dependent Releases**  
Require cryptographic proof of off-chain events (payment receipts, attestations) before enabling spend.

**Coordinated Multi-Party Operations**  
Ensure all parties have provided required evidence before any signing occurs. Atomic "all-or-nothing" precondition satisfaction.

### 1.3 When to Use BPC (Normative Guidance)

BPC SHOULD be used when **all** of the following are true:

- Settlement occurs on Bitcoin L1 using standard transactions
- The decision to proceed can be determined *before* transaction construction
- Required conditions can be expressed as verifiable evidence (time, consent, approvals, structure)
- Failure to meet conditions MUST result in refusal, not degraded execution
- An auditable record of authorization decisions is required

Typical conformant use cases: OTC settlement, policy-governed treasury, time-bounded releases, multi-party approval workflows, pre-authorized settlement pipelines.

### 1.4 Scope Definition

**This specification defines:**
- Deterministic evaluation of preconditions prior to transaction construction
- Fail-closed refusal when evidence, time, or bindings are missing or invalid
- Prevention of broadcast-valid spend artifacts before authorization
- Cryptographically verifiable audit trails for authorization decisions
- Compatibility with standard Bitcoin L1 settlement

**This specification does not define:**
- Consumer-side wallet UX or key management
- Application-level business logic beyond authorization
- Bitcoin consensus or transaction validity rules
- On-chain execution, covenants, or Script extensions
- Miner behavior, inclusion guarantees, or censorship resistance

### 1.5 Boundary Conditions (Normative)

**BPC MUST NOT be used when any of the following are required:**

- On-chain execution of conditional logic
- Enforcement of conditions *after* broadcast
- Subjective judgment, interpretation, or dispute resolution
- Guarantees of miner inclusion, ordering, or censorship resistance
- Continuous or stateful execution across multiple transactions
- Automatic or unattended signing without explicit authorization

Attempting to use BPC in these scenarios is a **category error** and results in non-conformant systems that provide misleading security guarantees.

**Common misinterpretations to avoid:**

BPC is not a smart contract language, virtual machine, or programmable execution environment. It is not an on-chain execution system or covenant mechanism. It does not modify Bitcoin consensus, Script semantics, or transaction validity rules. It is not a replacement for PSBT, wallet signing logic, or key custody systems. It is not a general-purpose automation framework, workflow engine, oracle network, truth arbitration system, or attestation authority.

Any implementation or documentation treating BPC as performing on-chain execution, enforcing consensus rules, or asserting external truth claims is **non-conformant** with this specification.

BPC provides **authorization refusal semantics only**. It determines whether a transaction *may be constructed*, not how it executes, how it is settled, or whether it is included in a block.

---

## 1.6 Security Model

### 1.6.1 Threat Model

BPC addresses **unauthorized or premature transaction construction**.

**Threats BPC prevents:**
- Construction of broadcast-valid transactions before required conditions are satisfied
- Replay of authorization decisions outside their validity window
- Substitution or mutation of transaction structure after authorization
- Execution proceeding under stale, ambiguous, or unverifiable time or evidence
- Partial execution or secret exposure during failed execution attempts
- Ambiguity-driven downgrade or "best effort" execution paths

**Threats BPC does not address:**
- Theft or compromise of signing keys outside the BPC-controlled environment
- Malicious or coerced signers acting against their own interests
- Miner censorship, reordering, fee manipulation, or reorgs
- Denial-of-service against networks, mirrors, or evaluators
- Side-channel attacks on cryptographic implementations
- Bugs or vulnerabilities in wallet software, operating systems, or hardware
- Disputes over intent, fairness, or contract interpretation
- Legal enforceability, remedies, or arbitration outcomes

These require secure key custody practices, robust wallet and OS security, Bitcoin's own consensus properties, and operational/legal controls external to this specification.

**Assumed attacker capabilities:**
- Observe network traffic and mempool state
- Attempt to replay or substitute authorization artifacts
- Attempt to present stale or partial evidence
- Attempt to exploit ambiguity in time or evaluation order
- Control or compromise individual mirrors or evaluators
- Exploit implementation bugs if present

**Assumed attacker limitations:**
- Cannot forge valid cryptographic signatures without key material
- Cannot cause deterministic evaluation to yield different results for identical inputs
- Cannot bypass refusal semantics without violating explicit conformance rules

### 1.6.2 Security Invariants (Normative)

The following security invariants MUST hold for any conformant BPC implementation. Violation of any invariant constitutes **critical non-conformance**.

---

#### Invariant 1 — No Pre-Authorization Spend Artifact

No broadcast-valid primary spend transaction or PSBT containing spend-authorizing material MUST exist before a corresponding `ALLOW` outcome is issued.

This includes:
- Fully formed transactions
- Partially signed PSBTs
- Recoverable signing material
- Any artifact that could be finalized into a valid spend without re-evaluation

---

#### Invariant 2 — Deterministic Evaluation

Given identical canonical inputs:
- `PreContractIntent`
- `EvidenceBundle`
- Verified Epoch Clock tick bytes
- Declared policy configuration

The evaluation result (ALLOW/DENY/LOCKED) MUST be identical across:
- Executions
- Machines
- Implementations
- Time (within validity windows)

Any divergence constitutes a correctness failure.

---

#### Invariant 3 — Fail-Closed on Ambiguity

If any required input is:
- Missing
- Stale
- Malformed
- Unverifiable
- Ambiguous

The system MUST refuse authorization.

There MUST be no fallback, default-allow, or "best effort" execution path.

---

#### Invariant 4 — Binding Integrity

An `ALLOW` outcome MUST be bound to:
- A specific intent (`intent_hash`)
- A specific contract context (`contract_id`, `session_id`)
- A specific transaction structure (`bound_psbt_hash`)

Any mutation or substitution after authorization MUST be detected and refused.

---

#### Invariant 5 — Single-Use Authorization

Each authorization outcome (`decision_id`) MUST be:
- Usable at most once
- Time-bounded by `valid_until_t`
- Permanently rejected after use or expiry

Replay or reuse MUST be detected and denied.

---

#### Invariant 6 — Time Authority Exclusivity

All time-based decisions MUST derive exclusively from verified Epoch Clock artifacts.

System clocks, NTP, DNS time, or inferred wall time MUST NOT influence:
- Expiry evaluation
- Freshness checks
- Authorization validity

If verified time is unavailable, the correct behavior is refusal.

---

#### Invariant 7 — Sovereign Signing

BPC MUST NOT compel signing.

The final act of signing remains under the control of the key holder. BPC provides refusal logic and authorization discipline only.

Any implementation that removes, bypasses, or obscures signer sovereignty is non-conformant.

---

These invariants define the **minimum security posture** of BPC. Implementations MAY provide stronger guarantees, but MUST NOT weaken or bypass any invariant defined above.

---

## 1.7 Implementation Classes (Informative)

This specification supports multiple conformant implementation classes. Not all implementations are required to implement all components, provided the security invariants in Section 1.6.2 are preserved.

The following classes are illustrative, not exhaustive.

---

#### Class A — Wallet-Only Enforcement

A wallet-only implementation:

- Evaluates BPC authorization locally
- Enforces refusal semantics before signing
- Constructs and signs transactions only after ALLOW
- Does not expose spend-authorizing artifacts pre-authorization

This class:
- Requires no external evaluator
- Is suitable for personal custody and single-operator use
- Preserves full signer sovereignty

Typical components:
- PreContractIntent evaluation
- Evidence verification
- Epoch Clock client
- PSBTTemplateCanonical binding
- Local replay protection

---

#### Class B — External Evaluator + Wallet Execution

A split implementation where:

- An external evaluator performs deterministic evaluation
- The evaluator issues signed ALLOW/DENY/LOCKED outcomes
- Wallets independently verify outcomes before signing
- Execution remains wallet-controlled

This class:
- Allows organizational separation of duties
- Supports policy centralization without custody centralization
- Requires no trust in the evaluator beyond signature verification

Typical components:
- External evaluation service
- Signed PreContractOutcome
- Wallet-side verification and execution gating
- Independent Epoch Clock verification on both sides

---

#### Class C — Multi-Evaluator Quorum

An implementation using multiple independent evaluators:

- Requires M-of-N evaluator agreement
- Aggregates multiple signed outcomes
- Proceeds only when quorum conditions are met
- Tolerates individual evaluator failure or compromise

This class:
- Reduces single-evaluator risk
- Is suitable for institutional or high-value operations
- Preserves deterministic evaluation by requiring identical inputs

Typical components:
- Multiple evaluator services
- Outcome aggregation logic
- Quorum policy enforcement
- Wallet-side verification of all bindings

---

#### Class D — Full-Stack Integrated Enforcement

A full-stack implementation combining:

- Intent creation
- Evaluation
- Execution
- Audit logging
- Policy management

This class:
- Is suitable for tightly controlled environments
- Still MUST preserve signer sovereignty
- MUST NOT collapse evaluation and signing authority

Typical components:
- Integrated BPC evaluator
- Wallet execution engine
- Audit ledger
- Policy management interface

---

#### Non-Conformant Classes

The following are **explicitly non-conformant**:

- Implementations that auto-sign without enforceable refusal
- Implementations that treat evaluator output as authoritative without verification
- Implementations that allow transaction construction before ALLOW
- Implementations that bypass PSBT template binding
- Implementations that collapse signer and policy authority without auditability

---

This section is informational and does not introduce new requirements. All implementation classes MUST satisfy the security invariants defined in Section 1.6.2.

---

## 1.8 Trust Boundary Between Host, Evaluator, and Signer (Normative)

BPC explicitly assumes that **host software is untrusted**.

A conformant implementation MUST enforce the following trust boundary:

1. **The signer is the final authority.**  
   A signing component (including a hardware wallet, secure enclave, or air-gapped signer) MUST independently verify that a valid `PreContractOutcome` authorizing the operation is present before producing any signature.

2. **Host software MUST NOT be trusted to enforce refusal.**  
   The host MAY perform evidence collection, evaluation, and Pre-Contract assembly, but its output MUST be treated as advisory until verified by the signer.

3. **A signer MUST refuse to sign if any of the following are true:**
   - No `PreContractOutcome` is presented
   - The `PreContractOutcome` signature is invalid
   - The `bound_intent_hash` does not match `PreContractIntent.intent_hash`
   - The `bound_session_id` or `bound_contract_id` do not match the active operation context
   - The outcome is expired (`current_t >= valid_until_t`)
   - The outcome is replayed (`decision_id` already used)
   - The outcome `decision` is not `ALLOW`
   - The `bound_psbt_hash` does not match the computed hash of the canonical template

4. **PSBT export prior to outcome binding is non-conformant.**  
   Any implementation that allows a spend-authorizing PSBT (or any artifact containing signatures or witness/final scripts for the primary spend) to be exported, persisted, or transmitted before `ALLOW` is issued is non-conformant.

5. **Bypass visibility requirement.**  
   If an implementation permits an explicit manual bypass (for example, raw PSBT export), it MUST:
   - Require an explicit, audited override action
   - Mark the resulting artifact as policy-violating
   - Disable claims of BPC conformance for that operation

This trust boundary preserves the core invariant:

> no broadcast-valid spend artifact exists until preconditions are satisfied.

---

## 2. Dependency and Authority Boundary

### 2.1 Bitcoin Boundary (Normative)

**Bitcoin remains the sole authority for transaction validity and settlement finality.**

BPC does not add on-chain logic and does not create Bitcoin-layer vulnerabilities. Transactions produced after BPC evaluation are standard Bitcoin transactions subject only to Bitcoin consensus rules.

BPC operates entirely off-chain in the construction phase. Once a transaction is broadcast, BPC has no further role. Enforcement is complete at the refusal-to-construct gate.

### 2.2 Epoch Clock v2 Boundary (Normative)

BPC uses Epoch Clock v2 for all time semantics.

**Pinned canonical profile_ref (MUST):**


```
"ordinal:439d7ab1972803dd984bf7d5f05af6d9f369cf52197440e6dda1d9a2ef59b6ebi0"
```

**Time Authority Philosophy:**

Epoch Clock does not create a new time oracle. It anchors time profile authority to a Bitcoin inscription and makes time artifacts cryptographically verifiable against that immutable anchor.

The profile is inscribed on Bitcoin (immutable), and ticks are signed with cryptographic signatures as defined by the pinned Epoch Clock profile (which MAY include post-quantum schemes).

Mirrors provide **verification redundancy**, not consensus. No single mirror is authoritative; clients validate locally.

Epoch Clock artifacts are **JCS Canonical JSON bytes** and:

* MUST be treated as **externally canonical opaque bytes**
* MUST NOT be re-encoded into CBOR or any other format
* MUST be fetched and accepted according to the mirror agreement protocol in Section 2.2.4
* MUST NOT use system clocks, NTP, DNS time, or wall clocks for any authority decision

On mirror divergence, unavailable ticks, or profile mismatch, the system MUST fail closed.

#### 2.2.1 Time Field Definition (Normative)

Epoch Clock `EpochTick` contains:

* `t` (uint): **Strict Unix Time** in seconds (1970-01-01T00:00:00Z, ignoring leap seconds)

All BPC time values are expressed as Unix seconds and MUST be compared against the verified Epoch Clock `t`.

Accordingly, BPC uses the suffix `_t` for Unix-seconds fields:

* `expiry_t`, `issued_t_hint`, `issued_t`, `valid_until_t`, `current_t`

For evaluation and execution, `current_t` MUST be derived exclusively from the verified Epoch Clock tick value `t` for the operation attempt (i.e. `current_t = EpochTick.t`). No other time source is permitted.

#### 2.2.2 Tick Reuse Window (Normative)

BPC permits limited reuse of the most recently validated Epoch Clock tick to support offline and partition-tolerant operation, while preventing indefinite operation on stale time.

Let:

- `tick_cached` be the most recently validated `EpochTick` (as canonical JCS JSON bytes)
- `validated_at_mono_ms` be the local **monotonic** timestamp (milliseconds) recorded at the moment `tick_cached` was validated

`current_t` is the authoritative Unix-seconds time value asserted by the most recently verified Epoch Clock tick.

`current_t` MUST be used only for evaluation and binding purposes.

`current_t` MUST NOT be used to infer time progression, determine freshness, or substitute for a new Epoch Clock tick.

**Reuse rule (MUST):**

1. Implementations MAY reuse `tick_cached` only while:

```
mono_now_ms - validated_at_mono_ms <= 900000
```

2. If the monotonic delta exceeds 900 seconds, the implementation MUST treat time as unavailable for all time-dependent operations and MUST fail closed.

3. If a monotonic clock is unavailable or cannot be trusted as monotonic, the implementation MUST treat time as unavailable and MUST fail closed.

**No authority claim:**

- The monotonic timer is used solely to enforce the maximum reuse duration
- It MUST NOT be used to compute, compare, or infer Unix time values

**Rationale:**  
Epoch Clock provides time authority; the monotonic timer enforces only a bounded reuse cutoff. This preserves deterministic refusal semantics during partitions without introducing wall-clock authority.

---

#### 2.2.3 Epoch Clock Failure Propagation (Normative)

BPC treats verifiable time as a hard security dependency for all time-bound operations.

1. If a valid Epoch Tick cannot be obtained and validated — including cases of mirror divergence, profile mismatch, invalid signature, or reuse-window expiry — the evaluator MUST emit `decision = "LOCKED"` with an appropriate `reason_code` in the `E_TICK_*` or `E_MIRROR_*` error families.

2. The executor MUST treat `decision = "LOCKED"` as refusal and MUST NOT construct, sign, or broadcast any transaction.

3. No local fallback is permitted. System clocks, NTP, DNS time, block timestamps, or application-provided time MUST NOT be used as substitutes for Epoch Clock artifacts.

This preserves a single interpretation: `LOCKED` represents fail-closed unavailability, while refusal is the execution posture.

#### 2.2.4 Mirror Agreement Protocol (Normative)

Epoch Clock ticks MUST be accepted only when sufficient independent mirror agreement is achieved.

This section defines the minimum protocol required to determine agreement.

##### Query Requirements

1. Implementations MUST query **at least three** independent Epoch Clock mirrors.
2. Mirrors MUST be queried using the pinned `profile_ref`.
3. Each mirror response MUST be validated independently for:
   - profile_ref match
   - JCS canonical encoding
   - Signature validity

##### Agreement Rule

1. A tick is considered **valid** only if **two or more mirrors return byte-identical `tick_bytes`**.
2. Agreement is defined strictly by **byte equality**, not by semantic equality of parsed fields.
3. If no such agreement is reached, the evaluator MUST fail closed with `E_MIRROR_DIVERGENCE`.

##### Partial Responses

1. If fewer than two mirrors return a response within the implementation-defined timeout window, the evaluator MUST fail closed with `E_TICK_MISSING`.
2. If responses are received but no byte-identical pair exists, the evaluator MUST fail closed with `E_MIRROR_DIVERGENCE`.

##### Retry Behavior

1. Implementations MAY query additional mirrors beyond the initial set.
2. Acceptance MUST occur only after meeting the agreement rule above.
3. If agreement cannot be reached before the reuse window expires, the evaluator MUST emit `decision = "LOCKED"`.

##### Authority Boundary

Mirror agreement provides **verification redundancy only**. Mirrors do not vote, elect leaders, or form consensus. All authority remains local to the verifier.

This protocol ensures deterministic, replay-safe time verification without introducing a new consensus layer.

### 2.3 Mempool Observation Disclaimer (Normative)

If implementations use mempool observation as evidence:

* It MUST be treated as **non-authoritative risk signal only**
* It MUST NOT be the sole basis for ALLOW
* Implementations MUST NOT make guarantees based on mempool visibility
* Mempool data is incomplete, adversarially influenceable, and not consensus-backed

**Rationale:** Mempool state is probabilistic and attackers control their own propagation behavior.

---

## 3. Architecture Overview

BPC defines a three-phase model:

1. **Intent** (non-authoritative, safe to publish)
2. **Evaluation** (deterministic precondition checks)
3. **Execution** (construct + sign + broadcast via wallet)

No executable transaction exists before ALLOW.

```
App / UI
→ PreContractIntent (what needs to happen)
→ EvidenceBundle (proof that conditions are met)
→ PreContractOutcome (ALLOW / DENY / LOCKED)
→ (only if ALLOW) PSBTTemplateCanonical finalized → PSBT constructed → signed → broadcast
```

### 3.1 Phases Explained

**Intent Phase:**
- Application declares what transaction it wants to construct
- Intent includes contract terms, expiry, required evidence types
- Intent is safe to share publicly, contains no secrets or spend-authorizing signatures
- Intent is content-addressed via `intent_hash`

**Evaluation Phase:**
- Evaluator receives intent + evidence bundle
- Verifies all evidence deterministically
- Checks time bounds via Epoch Clock
- Validates approvals, proofs, policy constraints
- Emits signed outcome with short validity window

**Execution Phase:**
- Executor receives intent + outcome + PSBTTemplateCanonical
- Verifies outcome signature and expiry
- Enforces all bindings (intent_hash, session_id, bound_psbt_hash)
- Constructs transaction only if ALLOW
- Signs and broadcasts via standard wallet flow

---

## 4. Normative Keywords

MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, MAY, OPTIONAL follow RFC 2119 meaning.

Additionally:

* **RECOMMENDED** indicates best practice that should be followed absent strong reason
* **NOT RECOMMENDED** indicates practice that should be avoided absent strong reason

---

## 5. Deterministic Encoding Rules

### 5.1 BPC-native Objects (Normative)

All BPC-native objects that are hashed or signed MUST be encoded using **Deterministic CBOR** (DetCBOR):

* Canonical key ordering (lexicographic by encoded byte value)
* Definite-length encoding only
* Unique map keys (no duplicates)
* No floats (use fixed-point integers represented as integers with defined scale)
* Smallest possible integer representation

### 5.2 Epoch Clock Objects (Normative)

Epoch Clock artifacts MUST be handled as **externally canonical**:

* Stored and compared as **opaque JCS JSON byte strings**
* Verified against the pinned profile_ref
* MUST NOT be re-serialized into CBOR by BPC implementations
* MUST NOT be parsed and re-emitted using non-JCS tooling

**Rationale:** Epoch Clock defines its own canonical encoding. Transcoding risks introducing divergence and breaking signature verification.

### 5.3 Hash Functions (Normative)

Unless otherwise specified:

* All hashes MUST use **SHA-256**
* Hash inputs MUST be the canonical DetCBOR encoding of the object
* Hash outputs are 32-byte values represented as byte strings

---

## 6. Core Objects

### 6.1 PreContractIntent (Non-authoritative)

Intent is safe to publish and MUST NOT contain:

* Signatures that authorize spending
* Spend-time secrets (private keys, signing material)
* Finalized raw transactions
* Any artifact that would make the primary spend broadcast-valid

**Fields (DetCBOR):**

```
PreContractIntent = {
  version: uint,                     ; schema version (currently 1)
  contract_id: bstr(16..32),         ; unique contract identifier
  issued_t_hint: uint / null,        ; OPTIONAL non-authoritative hint (unix seconds)
  expiry_t: uint,                    ; unix seconds, evaluated using verified EpochTick.t
  session_id: bstr(16..32),          ; binds evaluation context
  intent_body: map,                  ; contract-specific payload
  intent_hash: bstr(32)              ; SHA-256(DetCBOR(intent_without_hash))
}
```

**Validation Rules (Normative):**

1. `intent_hash` MUST recompute correctly as SHA-256(DetCBOR(intent with intent_hash field removed))
2. `version` MUST equal 1 (reject unknown versions)
3. `contract_id` MUST be unique within system scope
4. `expiry_t` MUST be enforced using verified Epoch Clock `t` only
5. If `issued_t_hint` is present and `expiry_t <= issued_t_hint`, evaluators SHOULD reject with `E_INTENT_INVALID_WINDOW` as structural sanity check

**Rationale for issued_t_hint:** Allows early rejection of obviously malformed intents without needing verified tick. Authority remains with verified time comparisons.

---

### 6.2 EpochTickBytes (Evidence)

Epoch tick evidence MUST be carried as:

```
EpochTickBytes = {
  profile_ref: tstr,                 ; MUST equal pinned canonical ref
  encoding: tstr,                    ; MUST equal "JCS-JSON"
  tick_bytes: bstr                   ; exact JCS canonical JSON bytes
}
```

**Validation Rules (Normative):**

1. `profile_ref` MUST equal `"ordinal:439d7ab1972803dd984bf7d5f05af6d9f369cf52197440e6dda1d9a2ef59b6ebi0"`
2. `encoding` MUST equal `"JCS-JSON"`
3. `tick_bytes` MUST be the exact JCS canonical JSON bytes of a valid EpochTick
4. Implementations MUST validate tick authenticity per Epoch Clock v2 rules:
   - Verify ML-DSA-65 signature over tick_bytes
   - Verify profile lineage and governance rules
   - Verify mirror agreement per Section 2.2.4
   - Enforce reuse window semantics as in Section 2.2.2

---

### 6.3 EvidenceBundle (Normative)

`EvidenceBundle` is the complete, canonical input set used for deterministic precondition evaluation.

An `EvidenceBundle` MUST include verifiable time evidence. There is no time-optional evaluation mode in BPC.

```
EvidenceBundle = {
  version: uint,                     ; schema version (currently 1)
  contract_id: bstr(16..32),         ; MUST match intent.contract_id
  intent_hash: bstr(32),             ; MUST match intent.intent_hash

  epoch_tick: EpochTickBytes,        ; REQUIRED

  approvals: [* Approval] / null,    ; OPTIONAL
  attestations: map / null,          ; OPTIONAL external attestations
  external_proofs: map / null,       ; OPTIONAL (receipts, oracle proofs)
  extra: map / null                  ; OPTIONAL extension point
}
```

**Approval Structure:**

```
Approval = {
  approver_id: tstr,                 ; identifier for approver (pubkey or role)
  role: tstr,                        ; role name for policy evaluation

  approval_t: uint,                  ; Unix seconds derived from Epoch Clock
                                     ; at approval creation time

  scope_hash: bstr(32),              ; binds approval to specific scope
  signature_input: bstr,             ; DetCBOR(ApprovalCommitment)
  sig: bstr,                         ; signature over signature_input
  sig_alg: tstr                      ; signature algorithm identifier
}

ApprovalCommitment = {
  intent_hash: bstr(32),
  contract_id: bstr(16..32),
  session_id: bstr(16..32),
  role: tstr,
  scope_hash: bstr(32),
  approval_t: uint
}
```

#### 6.3.1 Approval Time Semantics (Normative)

Approvals MAY include a time marker to constrain replay and ordering. All approval timing semantics are evaluated **exclusively** against verified Epoch Clock time.

##### Time Authority Requirement

1. `approval_t` MUST be derived from a **verified Epoch Clock tick** at the time the approval is created.
2. Implementations MUST NOT derive `approval_t` from:
   - System clocks
   - Wall clocks
   - NTP
   - Inferred or approximate time sources

##### Approval Validation Rules (Normative)

An approval is valid only if **all** of the following hold:

1. The signature verifies over `ApprovalCommitment`.
2. `approval_t <= current_t` (approval is not from the future).
3. If `issued_t_hint` is present in the intent:
   - `approval_t >= issued_t_hint` MUST hold as a structural sanity check.
4. `approval_t < intent.expiry_t` MUST hold.
5. All binding fields (`intent_hash`, `contract_id`, `session_id`, `role`, `scope_hash`) MUST match the active evaluation context.

If any rule fails, the evaluator MUST treat the approval as invalid and proceed according to policy (typically `DENY` or `LOCKED`).

##### Rationale

Approvals are security-critical evidence. Allowing approvals to carry timestamps derived from local clocks would reintroduce nondeterminism and downgrade risk. By binding approval time to Epoch Clock authority, approval ordering and replay constraints remain deterministic and verifiable across implementations.

#### Approval Signature Algorithms (Normative)

Implementations MUST:

- Declare supported `sig_alg` values in conformance metadata
- Fail closed on unknown `sig_alg`

Implementations SHOULD support at minimum:

- `"ECDSA-secp256k1-SHA256"`
- `"Ed25519"`

Implementations MAY additionally support:

- `"ML-DSA-65"`

#### Approval Signature and Binding Validation Rules (Normative)

1. `signature_input` MUST equal DetCBOR(ApprovalCommitment) with fields matching the Approval.
2. `sig` MUST verify under `approver_id` pubkey using `sig_alg`.
3. `scope_hash` MUST match expected scope for the approval (contract-specific).
4. Approvals are evidence only. Policy determines whether approvals are sufficient.

#### Validation Rules (Normative)

1. `version` MUST equal 1. Unknown versions MUST be rejected.
2. `contract_id` MUST equal `PreContractIntent.contract_id`.
3. `intent_hash` MUST equal `PreContractIntent.intent_hash`.
4. `epoch_tick` MUST be present and MUST be validated per Section 2.2.
5. If `epoch_tick` is missing, invalid, stale, or unverifiable, the evaluator MUST emit `decision = "LOCKED"`.

There is no conformant evaluation path that omits verifiable time.

#### Rationale

All BPC intents include explicit expiry semantics via `expiry_t`. Allowing evaluation without a verified time source would introduce ambiguity, downgrade risk, and divergent behavior across implementations.

Accordingly, time is treated as a mandatory security input rather than an optional predicate.

---

### 6.4 PSBTTemplateCanonical (Normative)

This object defines the canonical transaction template that is bound into `PreContractOutcome.bound_psbt_hash`.

A `PSBTTemplateCanonical` represents a **non-spendable transaction skeleton**. It MUST exclude all spend-authorizing material and MUST be stable across implementations.

```
PSBTTemplateCanonical = {
  version: uint,                     ; MUST be 1
  inputs: [* PSBTInputTemplate],
  outputs: [* PSBTOutputTemplate],
  locktime: uint,
  fee_rate_sat_vb: uint / null       ; OPTIONAL, but if present it is BINDING
}

PSBTInputTemplate = {
  txid: bstr(32),
  vout: uint,
  sequence: uint
}

PSBTOutputTemplate = {
  script_pubkey: bstr,
  amount_sats: uint
}
```

#### Structural Rules (Normative)

1. `version` MUST equal 1. Unknown versions MUST be rejected.
2. `inputs` and `outputs` MUST be non-empty.
3. `sequence` values fully define replaceability semantics. Implementations MUST NOT use an independent boolean or inferred flag to represent replaceability.
4. If `fee_rate_sat_vb` is present, it MUST be treated as binding input to transaction construction.
5. If fee is not intended to be bound, `fee_rate_sat_vb` MUST be null.
6. Implementations MUST NOT treat a non-null `fee_rate_sat_vb` as advisory or informational.

#### Ordering Rules (Normative)

1. `inputs` MUST be sorted lexicographically by `(txid, vout)` where:
   - `txid` is compared as raw bytes
   - `vout` is compared as an unsigned integer when `txid` matches

2. `outputs` MUST be sorted lexicographically by `(amount_sats, script_pubkey)` where:
   - `amount_sats` is compared ascending
   - `script_pubkey` is compared as raw bytes when amounts match

#### Exclusion Rules (Normative)

`PSBTTemplateCanonical` MUST NOT include:

- Partial signatures
- Final scripts or witnesses
- Unknown or proprietary PSBT fields
- `non_witness_utxo` or `witness_utxo` data
- Any key material or secrets
- Any field whose value may vary across implementations without an explicit canonical rule

If any excluded material is present in the template input provided to hashing, the implementation MUST refuse with `E_PSBT_TEMPLATE_NONCANONICAL`.

#### Binding Hash (Normative)

```
bound_psbt_hash = SHA-256(DetCBOR(PSBTTemplateCanonical))
```

### 6.4.1 PSBTTemplateCanonical Enforcement Rules (Normative)

A `PSBTTemplateCanonical` represents a **non-spendable transaction skeleton** that MAY be evaluated but MUST NOT be signed or broadcast.

The executor and signer MUST enforce the following rules prior to execution.

#### 1. Spend-Authorization Prohibition

A canonical PSBT template MUST contain **no spend-authorizing material**.

Specifically, it MUST NOT include:

- Signatures of any kind
- Witness data
- Finalized scripts
- Key paths or script paths indicating finalization
- Proprietary, vendor-specific, or unknown PSBT fields

Presence of any such material MUST result in refusal with `E_PSBT_TEMPLATE_NONCANONICAL`.

#### 2. Deterministic Ordering Enforcement

Implementations MUST enforce deterministic ordering exactly as follows:

- Inputs sorted lexicographically by `(txid, vout)`
- Outputs sorted lexicographically by `(amount_sats, script_pubkey)`
- All internal maps sorted per canonical DetCBOR rules

Any deviation MUST result in refusal.

#### 3. Fee Binding Semantics

Fee handling MUST be **unambiguous**.

- If `fee_rate_sat_vb` is present in `PSBTTemplateCanonical`, it MUST be treated as binding input to transaction construction.
- If fee is not intended to be bound, `fee_rate_sat_vb` MUST be null.
- Implementations MUST NOT interpret any fee-related field as advisory, informational, or best-effort.

Mixed fee semantics are forbidden.

#### 4. Canonical Encoding Stability

The canonical DetCBOR encoding of `PSBTTemplateCanonical` MUST be byte-stable.

- Decode → encode cycles MUST produce byte-identical output.
- Two independent implementations given the same semantic input MUST produce identical canonical bytes.

Failure to meet this requirement MUST result in refusal.

#### 5. Enforcement Outcome

Any violation of the rules in this section MUST result in refusal with `E_PSBT_TEMPLATE_NONCANONICAL`.

There MUST be no fallback, downgrade, or partial acceptance behavior.

### 6.4.2 Reference Canonicalizer Guidance (Informative)

Implementations SHOULD provide:

- A reference PSBT canonicalizer
- Canonicalization test vectors
- A conformance mode that verifies:
  - Canonical bytes
  - Template hash stability
  - Rejection of non-canonical templates

PSBT canonicalization errors are expected to be the primary source of implementation defects. The correct failure mode is deterministic refusal, not fallback behavior.

### 6.4.3 Replace-By-Fee (RBF) Semantics (Normative)

Replace-by-Fee (RBF) behavior in BPC is determined **exclusively** by input `sequence` values contained in `PSBTInputTemplate`.

1. **Authoritative definition**
   - An input is considered replaceable iff its `sequence` value is less than `0xFFFFFFFE`.
   - An input is considered non-replaceable iff its `sequence` value is greater than or equal to `0xFFFFFFFE`.

2. **No independent flags**
   - Implementations MUST NOT use a separate boolean, policy flag, or inferred heuristic to represent RBF state.
   - Any independent representation of RBF outside input `sequence` values is non-conformant.

3. **Binding requirement**
   - Input `sequence` values are part of the canonical template and are therefore bound by `bound_psbt_hash`.
   - Any mutation of `sequence` values after authorization MUST be detected and refused as a binding violation.

4. **Consistency enforcement**
   - If an implementation derives an RBF state that conflicts with the input `sequence` values, it MUST refuse with `E_PSBT_TEMPLATE_NONCANONICAL`.

This rule ensures that replaceability semantics are fully deterministic, Bitcoin-native, and unambiguous across implementations.

---

### 6.5 PreContractOutcome (Authoritative)

Outcome is the signed result of evaluation.

```
PreContractOutcome = {
  version: uint,                     ; schema version (currently 1)
  decision_id: bstr(16..32),         ; single-use decision identifier
  decision: tstr,                    ; "ALLOW" / "DENY" / "LOCKED"
  reason_code: tstr,                 ; standardized error/success code

  bound_intent_hash: bstr(32),       ; binds to specific intent
  bound_contract_id: bstr(16..32),   ; binds to specific contract
  bound_session_id: bstr(16..32),    ; binds to evaluation session

  bound_psbt_hash: bstr(32),         ; REQUIRED

  issued_t: uint,                    ; unix seconds from verified EpochTick.t
  valid_until_t: uint,               ; unix seconds, short validity window

  signature_input: bstr,             ; DetCBOR(OutcomeCommitment)
  sig: bstr,                         ; signature over signature_input
  sig_alg: tstr,                     ; signature algorithm
  evaluator_id: tstr                 ; identifier for evaluator (pubkey)
}

OutcomeCommitment = {
  decision_id: bstr(16..32),
  decision: tstr,
  reason_code: tstr,
  bound_intent_hash: bstr(32),
  bound_contract_id: bstr(16..32),
  bound_session_id: bstr(16..32),
  bound_psbt_hash: bstr(32),
  issued_t: uint,
  valid_until_t: uint
}
```

**Decision Values (Normative):**

* `"ALLOW"` - All preconditions satisfied, transaction construction permitted
* `"DENY"` - Preconditions failed, will not be satisfied with current evidence (stable rejection)
* `"LOCKED"` - System in fail-closed state, cannot evaluate (retry after resolution)

**Outcome Validation Rules (Normative):**

1. `decision_id` MUST be accepted at most once for execution (replay protection)
2. If `current_t >= valid_until_t`, executor MUST refuse with `E_OUTCOME_EXPIRED`
3. All binding fields MUST match corresponding intent/session/template values
4. `signature_input` MUST equal DetCBOR(OutcomeCommitment)
5. `sig` MUST verify under `evaluator_id` pubkey using `sig_alg`
6. `bound_psbt_hash` is REQUIRED and MUST be enforced by executors

**Validity Window Guidance (Normative):**

* `valid_until_t` SHOULD be set to `issued_t + 720` (12 minutes)
* Implementations SHOULD NOT accept validity windows longer than 1800 seconds (30 minutes)

### 6.5.1 Multi-Evaluator Quorum Semantics (Normative)

This section applies only to deployments using multiple independent evaluators
(Class C implementations as described in Section 1.7).

#### Input Consistency Requirement

All evaluators participating in a quorum MUST evaluate **byte-identical inputs**.

Specifically, each evaluator MUST receive:

- the same `PreContractIntent` (verified via `intent_hash`),
- the same `EvidenceBundle`, including **byte-identical** `epoch_tick.tick_bytes`,
- the same canonical `PSBTTemplateCanonical` (verified via `bound_psbt_hash`).

If any evaluator receives non-identical inputs, quorum evaluation is
**non-conformant** and MUST NOT proceed.

#### Outcome Production

1. Each evaluator MUST produce an independent `PreContractOutcome`.
2. Each `PreContractOutcome` MUST use a **unique** `decision_id`.
3. All outcomes MUST be evaluated against the same `current_t` derived from the
   verified Epoch Clock tick.

#### Quorum Satisfaction Rules

Quorum is satisfied only if **M evaluators** emit  
`decision = "ALLOW"` with **all** of the following fields identical:

- `bound_intent_hash`
- `bound_contract_id`
- `bound_session_id`
- `bound_psbt_hash`

Quorum outcomes MUST all be **unexpired at execution time**:

```
current_t < valid_until_t
```

for **each** outcome participating in the quorum.

#### Decision Precedence and Aggregation

The executor MUST aggregate evaluator outcomes using the following strict
precedence rules:

1. **DENY precedence**  
   If **any evaluator emits `decision = "DENY"`**, the executor MUST treat the
   overall result as `DENY`.

2. **LOCKED handling**  
   If no `DENY` is present and **any evaluator emits `decision = "LOCKED"`**, the
   executor MUST treat the overall result as `LOCKED`.

3. **ALLOW quorum**  
   Only if no `DENY` or `LOCKED` outcomes are present, and quorum satisfaction
   rules are met, MAY the executor proceed with `ALLOW`.

There MUST be no fallback, tie-breaking, or “best effort” behavior.

#### Binding Divergence Handling

If evaluators emit outcomes with **divergent binding fields**
(e.g. differing `bound_psbt_hash` or `bound_session_id`), the executor MUST:

- treat the result as **fail-closed**,
- refuse execution,
- and MUST NOT attempt to reconcile or select among outcomes.

Such divergence indicates an implementation error or evaluator compromise.

#### Execution with Quorum

When quorum is used, the executor MUST:

1. Independently verify the signature of **each** `PreContractOutcome`.
2. Enforce replay protection for **all** `decision_id` values used in the quorum.
3. Proceed to execution only if quorum conditions are met and the aggregated
   decision is `ALLOW`.

#### Determinism Violation Detection

If evaluators receive identical inputs but emit different decisions or bindings,
this constitutes a **determinism violation**.

In such cases:

- the system MUST fail closed,
- execution MUST NOT proceed,
- operators SHOULD be alerted.

No automatic resolution or majority voting is permitted.


### 6.6 Decision Mapping: DENY vs LOCKED (Normative)

To ensure consistent behavior across implementations, evaluators MUST apply the following decision mapping rules.

#### 1. DENY — Definitive Failure

`DENY` MUST be used for **stable, definitive failures** where the intent and evidence set, as provided, cannot become valid without modification.

Examples include, but are not limited to:

- Intent expiry
- Binding mismatches (intent, session, contract, or template)
- Invalid or unverifiable signatures
- Failed policy constraints
- Structurally invalid or malformed evidence
- Approvals that do not satisfy required roles or thresholds

A `DENY` outcome permanently invalidates the associated intent for execution. Retrying the same intent after `DENY` is non-conformant.

#### 2. LOCKED — Fail-Closed Unavailability

`LOCKED` MUST be used for **fail-closed unavailability** where required dependencies cannot be safely evaluated at the time of assessment.

Examples include, but are not limited to:

- Inability to obtain or validate a fresh Epoch Clock tick
- Mirror divergence or profile mismatch
- Required evidence producers being unavailable
- Reuse-window expiry during offline or partitioned operation

A `LOCKED` outcome indicates that evaluation cannot proceed safely at the current time and that retry MAY be possible once the blocking condition is resolved.

#### 3. Execution Posture

Both `DENY` and `LOCKED` MUST result in refusal at execution time.

The distinction exists solely to differentiate:

- Permanent rejection (`DENY`)
- Temporary unavailability (`LOCKED`)

Executors MUST NOT construct, sign, or broadcast any transaction unless `decision = "ALLOW"`.

This mapping prevents divergent interpretations of failure states and guarantees uniform refusal semantics across all conformant implementations.

---

## 7. Execution Gate

The Execution Gate ensures:

* No broadcast-valid transaction exists before ALLOW
* Finalization is attempt-scoped and atomic (construct → sign → broadcast)
* Failures burn attempt-scoped secrets/material and require a new attempt
* All bindings are enforced at execution time

### 7.1 Required Interface (Normative)

Implementations MUST provide an executor with this semantic interface:

```python
class BPCExecutor:
    def execute(
        self,
        intent: PreContractIntent,
        outcome: PreContractOutcome,
        template: PSBTTemplateCanonical,
        current_t: int
    ) -> ExecutionResult:
        """
        MUST refuse if:
        - outcome.decision != "ALLOW"
        - outcome expired (current_t >= valid_until_t)
        - outcome replayed (decision_id seen before)
        - any binding mismatch exists
        - SHA-256(DetCBOR(template)) != outcome.bound_psbt_hash
        - template violates canonical rules (E_PSBT_TEMPLATE_NONCANONICAL)

        MUST NOT broadcast unless all checks pass.
        """
```

### 7.2 PSBT Template Binding (Normative)

Executor MUST:

1. Compute `SHA-256(DetCBOR(PSBTTemplateCanonical))`
2. Compare against `outcome.bound_psbt_hash`
3. REFUSE execution with `E_PSBT_TEMPLATE_MISMATCH` on mismatch
4. Construct transaction only from the verified template

### 7.2.1 Final Transaction Skeleton Binding (Normative)

After PSBT finalization and **before broadcast**, the executor MUST verify that the final transaction preserves the canonical structure authorized by `PSBTTemplateCanonical`.

#### Required Preservation

The following properties MUST match the authorized template exactly:

1. **Inputs**
   - Input count MUST match
   - Input ordering MUST match
   - Each input's `(txid, vout, sequence)` MUST match the template

2. **Outputs**
   - Output count MUST match
   - Output ordering MUST match
   - Each output's `(amount_sats, script_pubkey)` MUST match the template

3. **Locktime**
   - Transaction `locktime` MUST match `PSBTTemplateCanonical.locktime`

4. **Replaceability**
   - Replaceability semantics MUST be determined solely by input `sequence` values and MUST match the template

#### Permitted Variations

The following differences are explicitly permitted and MUST NOT cause refusal:

- Witness stack contents (signatures, preimages, annex data)
- `scriptSig` content for legacy or wrapped-SegWit inputs
- Transaction weight and vsize differences resulting from variable-length signatures

#### Verification Method

1. The executor MUST extract the **unsigned transaction skeleton** from the finalized PSBT.
2. The executor MUST compare the extracted skeleton against `PSBTTemplateCanonical`.
3. If any required preservation rule is violated, the executor MUST refuse with `E_FINAL_TX_TEMPLATE_DIVERGENCE`.

#### Rationale

`bound_psbt_hash` authorizes a specific transaction structure, not a best-effort approximation. This check prevents post-authorization mutation while allowing standard Bitcoin signature variability.

---

### 7.3 ExecutionResult (Normative)

`ExecutionResult` records the outcome of an execution attempt following a valid `PreContractOutcome`. It provides an auditable record of whether execution proceeded, was refused, or failed.

```
ExecutionResult = {
  version: uint,                    ; schema version (currently 1)
  execution_id: bstr(16..32),       ; unique execution attempt identifier
  decision_id: bstr(16..32),        ; links execution to authorization outcome
  contract_id: bstr(16..32),        ; links execution to contract instance
  intent_hash: bstr(32),            ; links execution to intent

  status: tstr,                     ; execution status
  txid: bstr(32) / null,            ; Bitcoin txid if broadcast occurred
  error: ErrorObject / null,        ; populated on failure or refusal

  result_t: uint                    ; Unix seconds derived from verified EpochTick.t
}

ErrorObject = {
  code: tstr,                       ; standardized error code (see Section 10)
  message: tstr,                    ; human-readable description
  detail: map / null                ; OPTIONAL implementation-specific context
}
```

#### Field Semantics

- `execution_id` uniquely identifies a single execution attempt and MUST NOT be reused.
- `decision_id` MUST refer to the `PreContractOutcome` used for this attempt.
- `intent_hash` and `contract_id` MUST match the corresponding fields in the intent.
- `result_t` MUST be derived exclusively from a verified Epoch Clock tick (`EpochTick.t`) obtained for the execution attempt. No local or inferred time source is permitted.

#### Status Values (Normative)

- `"NOT_SENT"`  
  Execution was aborted before any broadcast attempt due to local refusal or validation failure.

- `"REFUSED"`  
  Execution was explicitly refused due to authorization, binding, expiry, or replay failure.

- `"SENT"`  
  A transaction was successfully broadcast to the Bitcoin network.

- `"CONFIRMED"`  
  The broadcast transaction has been observed as included in a Bitcoin block. Confirmation tracking is OPTIONAL and outside BPC's core enforcement.

- `"FAILED"`  
  Execution was attempted but failed due to signing, construction, or broadcast error.

#### Error Handling Rules (Normative)

- `error` MUST be present when `status` is `"FAILED"` or `"REFUSED"`.
- `error` MUST be `null` when `status` is `"SENT"` or `"CONFIRMED"`.

#### ExecutionResult Invariants (Normative)

- An `ExecutionResult` MUST be produced for every execution attempt.
- An execution attempt MUST NOT transition from `"REFUSED"` or `"FAILED"` to `"SENT"`.
- A transaction MUST NOT be broadcast unless the associated outcome decision is `"ALLOW"` and all binding checks have passed.
- `result_t` MUST NOT be derived from local system time, inferred clocks, or wall time sources.

This object exists solely for auditability and operational clarity. It does not introduce new authority, guarantees, or enforcement semantics beyond those defined elsewhere in this specification.

### 7.4 Retry, Re-Evaluation, and State Discipline (Normative)

1. **DENY is terminal for an intent.**  
   A `DENY` outcome MUST permanently invalidate the associated intent for execution.

2. **Retry requires a new intent.**  
   Any retry MUST use:
   - A new `contract_id` or a new attempt-scoped `operation_id` within `intent_body`
   - A fresh evidence set
   - A new evaluation
   - A new `decision_id`

3. **LOCKED states**  
   When the system is `LOCKED` (or otherwise fail-closed due to time/evidence ambiguity), retries MUST be suppressed until:
   - Fresh verifiable time is available
   - Required predicates are satisfied
   - Policy permits resumption

This prevents replay and cross-attempt state confusion.

---

## 8. Epoch Clock v2 Compliance Requirements (Normative)

Any BPC implementation claiming conformance MUST implement:

- profile pinning to the canonical `profile_ref`,
- tick fetching and acceptance per the mirror agreement protocol in Section 2.2.4,
- treatment of tick bytes as opaque JCS Canonical JSON,
- no fallback to system clocks, NTP, DNS, or wall time,
- fail-closed behavior on divergence or unavailability,
- reuse window handling as specified in Section 2.2.2.

Failure to satisfy any requirement above constitutes non-conformance.

---

## 9. Deterministic Evaluation Algorithm (Normative)

Given identical canonical inputs, evaluators MUST produce identical outputs.

**Core Principle:** Same Intent + Same Evidence + Same Verified Tick = Same Outcome

### 9.1 Evaluation Steps (Normative)

Evaluators MUST perform these steps in order:

**Step 1: Intent Validation**

* Validate structure
* Recompute `intent_hash`
* DENY on mismatch (`E_INTENT_HASH_MISMATCH`) or invalid window (`E_INTENT_INVALID_WINDOW`)

**Step 2: Evidence Binding**

* Verify `EvidenceBundle.intent_hash == intent.intent_hash`
* Verify `EvidenceBundle.contract_id == intent.contract_id`
* DENY on mismatch (`E_EVIDENCE_INTENT_MISMATCH`)

**Step 3: Epoch Clock Validation**

* Validate pinned profile_ref
* Validate JCS-JSON encoding
* Verify signature over tick bytes
* Verify mirror agreement per Section 2.2.4
* LOCKED on failure with `E_TICK_*` or `E_MIRROR_DIVERGENCE`

**Step 4: Time Authority**

* Derive `current_t` from verified `EpochTick.t`
* No other time source permitted

**Step 5: Expiry Check**

* If `current_t >= intent.expiry_t` → DENY (`E_INTENT_EXPIRED`)

**Step 6: Preconditions**

* Approvals (9.1.1)
* External proofs (9.1.2)
* Policy constraints (9.1.3)

**Step 7: Template Binding**

* Construct canonical `PSBTTemplateCanonical`
* Compute `bound_psbt_hash`

**Step 8: Replay Guards**

* Enforce decision_id uniqueness
* Track intent execution if configured

**Step 9: Outcome Emission**

* Emit signed `PreContractOutcome` with short validity window

### 9.1.1 Approval Evaluation (Normative)

For each required approval role:

1. Find matching Approval with correct role
2. Verify `signature_input == DetCBOR(ApprovalCommitment)`
3. Verify commitment binds to intent_hash, contract_id, session_id
4. Verify signature under approver_id and sig_alg
5. Verify `approval_t <= current_t`

DENY on failure: `E_APPROVALS_MISSING` / `E_APPROVAL_INVALID`

### 9.1.2 External Proof Evaluation (Normative)

Proof verification MUST be deterministic and MUST NOT require network calls.

DENY on failure: `E_PROOF_INVALID` / `E_PROOF_MISSING`

### 9.1.3 Policy Constraint Evaluation (Normative)

Policy evaluation MUST be deterministic.

DENY on failure: `E_POLICY_DENY`

### 9.2 Replay Guards (Normative)

Implementations MUST maintain durable state:

* `seen_decision_ids` MUST survive restarts
* Reuse MUST refuse (`E_OUTCOME_REPLAYED`)

Pruning MUST use verified time only and SHOULD retain entries beyond expiry for safety.

---

## 10. Error Codes (Normative)

### 10.1 Epoch Clock Errors

* `E_TICK_MISSING`
* `E_TICK_PROFILE_MISMATCH`
* `E_TICK_CANONICAL_INVALID`
* `E_TICK_SIG_INVALID`
* `E_MIRROR_DIVERGENCE`
* `E_TICK_STALE`
* `E_TICK_LINEAGE_INVALID`

### 10.2 Structure and Binding Errors

* `E_INTENT_HASH_MISMATCH`
* `E_EVIDENCE_INTENT_MISMATCH`
* `E_SESSION_MISMATCH`
* `E_INTENT_EXPIRED`
* `E_INTENT_INVALID_WINDOW`
* `E_OUTCOME_EXPIRED`
* `E_OUTCOME_REPLAYED`
* `E_OUTCOME_NOT_ALLOW`
* `E_PSBT_TEMPLATE_MISMATCH`
* `E_PSBT_TEMPLATE_NONCANONICAL`
* `E_INTENT_ALREADY_EXECUTED`

### 10.3 Contract Evaluation Errors

* `E_POLICY_DENY`
* `E_APPROVALS_MISSING`
* `E_APPROVAL_INVALID`
* `E_PROOF_INVALID`
* `E_PROOF_MISSING`
* `E_INTERNAL_ERROR`

### 10.4 Execution Errors

* `E_BROADCAST_FAILED`
* `E_INSUFFICIENT_FEES`
* `E_SIGNING_FAILED`
* `E_PSBT_CONSTRUCTION_FAILED`
* `E_FINAL_TX_TEMPLATE_DIVERGENCE`

---

## 11. Evaluator Trust and Operational Models (Normative)

The evaluator is not a truth oracle. It verifies evidence.

The evaluator signature means:

> "Given these bytes, under these rules, the deterministic decision is X."

It does not mean external facts are true.

---

## 12. Conformance Targets

### 12.1 BPC/Core

A conformant implementation MUST:

1. Implement schemas and validation rules
2. Enforce DetCBOR
3. Enforce Epoch Clock v2 handling rules
4. Fail closed on ambiguity
5. Enforce decision_id single-use
6. Enforce PSBT template binding using PSBTTemplateCanonical

### 12.2 BPC/Bitcoin (Execution Profile)

Additionally MUST:

* Refuse signing/broadcast on any binding mismatch
* Produce ExecutionResult artifacts
* Implement replay-guard persistence

---

## 13. Determinism Validation (Normative)

Implementations MUST provide test vectors proving:

* Same inputs → same outputs across implementations
* Same inputs → same outputs across platforms
* Output hashes stable when signatures removed

---

## 14. Mandatory Test Vectors (Normative)

Conformant implementations MUST pass the BPC reference test vectors when published.

Implementations MUST ship test vectors demonstrating:

1. Basic ALLOW
2. Profile mismatch
3. Mirror divergence
4. Stale tick beyond reuse window
5. Intent expiry
6. Outcome replay
7. Outcome expiry
8. Intent hash mismatch
9. Session mismatch
10. Missing approvals
11. Invalid approval signature
12. PSBT template mismatch
13. PSBT template non-canonical rejection
14. Cross-implementation equivalence

---

## 15. Worked Example: OTC Delivery-vs-Payment (DvP)

(Example uses `*_t` Unix seconds fields and binds outcome to a canonical PSBT template.)

---

## 16. Capabilities Enabled by BPC

* Deterministic off-chain evaluation
* Enforceable preconditions before transaction existence
* Signed audit trail for allow/deny
* Standard Bitcoin settlement

---

## 17. Comparison to Existing Approaches (Informative)

BPC occupies: **deterministic off-chain evaluation + standard Bitcoin settlement**.

---

## 18. Privacy Considerations (Informative)

BPC does not provide privacy by default. Add privacy via encrypted intent bodies, ZK proofs, or evaluator quorum.

---

## 19. Fee Market Considerations (Informative)

Fee volatility is handled operationally using short validity windows, fee buffers, and optional RBF policies (expressed in template).

---

## 20. Future Extensions (Informative)

ZK evidence, multi-evaluator quorum, cross-chain bridge discipline, and richer proof types.

---

## 21. Implementation Checklist

* [ ] DetCBOR encoder/decoder
* [ ] Epoch Clock v2 client (mirror agreement per Section 2.2.4, JCS bytes, pinned profile)
* [ ] Evaluator with deterministic ordering
* [ ] PSBTTemplateCanonical hashing and enforcement
* [ ] Durable replay guard store
* [ ] Executor integration with wallet signing/broadcast
* [ ] Test vectors and determinism harness

---

## 22. Naming and Positioning

Pre-Contracts execute *before* a Bitcoin transaction exists.

BPC is not an L2 and does not modify Bitcoin.

---

## 23. FAQ

**Q: Isn't this just smart contracts?**  
A: No. Smart contracts execute on-chain. BPC executes off-chain and only produces standard Bitcoin transactions. No new opcodes, no VM, no consensus changes.

**Q: Why not just use Bitcoin Script?**  
A: Bitcoin Script is limited and doesn't support complex preconditions. BPC complements Script by handling preconditions off-chain while settlement remains on-chain.

**Q: Doesn't this create a trusted oracle?**  
A: No. Epoch Clock is decentralized (Bitcoin-inscribed profile, independent mirrors). Time authority derives from Bitcoin's immutability, not a single entity.

**Q: Can someone bypass BPC and just sign directly?**  
A: Yes — but then they violate their own policy. BPC doesn't stop signing; it enables wallets to refuse signing unless conditions are met. Sovereignty remains with the signer.

**Q: Why not use Lightning or another L2?**  
A: BPC is for on-chain settlement with preconditions. Lightning is for off-chain payment channels. Different use cases.

**Q: Isn't this overengineering?**  
A: For simple transactions, yes. For institutional custody, OTC, and policy-enforced multisig, BPC provides necessary rigor.

**Q: Does this require running new infrastructure?**  
A: Only if you want to run your own Epoch Clock mirror. Most users can use public mirrors just like they use public Bitcoin nodes.

---

## 24. Security Considerations

Template substitution prevented by mandatory template binding. Replay prevented by decision_id single-use and durable replay store. Time manipulation prevented by Epoch Clock v2.

---

## Appendix A — Boundary Clarifications (Normative)

The following examples illustrate **non-conformant interpretations or uses** of BPC. Any implementation or documentation exhibiting these patterns MUST be considered non-compliant with this specification.

### A.1 Treating BPC as an On-Chain Contract System

**Non-conformant behavior:**  
- Claiming BPC "executes contracts on Bitcoin"
- Encoding BPC logic into Bitcoin Script
- Expecting miners to enforce BPC rules

**Why non-conformant:**  
BPC performs no on-chain execution and defines no Script semantics. All evaluation occurs strictly off-chain prior to transaction construction.

---

### A.2 Treating BPC Outcomes as Settlement Guarantees

**Non-conformant behavior:**  
- Claiming ALLOW guarantees transaction inclusion
- Claiming ALLOW prevents censorship or reorgs
- Claiming ALLOW enforces ordering or timing on miners

**Why non-conformant:**  
Bitcoin miners remain the sole arbiters of inclusion and ordering. BPC authorization ends at transaction construction.

---

### A.3 Treating Evaluators or Mirrors as Trusted Authorities

**Non-conformant behavior:**  
- Treating a single evaluator as authoritative truth
- Treating a single Epoch Clock mirror as authoritative time
- Skipping local verification in favor of "trusted servers"

**Why non-conformant:**  
BPC requires local verification, deterministic evaluation, and fail-closed behavior. Mirrors provide availability, not authority.

---

### A.4 Treating BPC as a Quantum-Safe Bitcoin Replacement

**Non-conformant behavior:**  
- Claiming BPC "makes Bitcoin quantum safe"
- Claiming BPC replaces secp256k1
- Claiming BPC prevents quantum attacks on on-chain keys

**Why non-conformant:**  
BPC does not modify Bitcoin consensus cryptography. Post-quantum mechanisms are optional extensions enforced via policy layers, not consensus.

---

### A.5 Treating BPC as a Dispute Resolution or Escrow System

**Non-conformant behavior:**  
- Expecting BPC to resolve disagreements
- Expecting BPC to judge intent, fairness, or correctness
- Expecting BPC to reverse or amend executed transactions

**Why non-conformant:**  
BPC evaluates evidence deterministically. It does not interpret intent, resolve disputes, or provide remedies.

---

### A.6 Treating BPC as Mandatory or Coercive

**Non-conformant behavior:**  
- Claiming BPC prevents users from signing transactions
- Claiming BPC enforces policy beyond user choice
- Claiming BPC removes user sovereignty

**Why non-conformant:**  
BPC enables refusal logic. The decision to sign always remains with the key holder.

---

## Appendix B — Evaluator Topology and Quorum Guidance (Informative)

BPC supports both single-evaluator and multi-evaluator deployments.

Implementers SHOULD note:

- Single-evaluator deployments introduce social and operational centralization risk
- Multi-evaluator quorum reduces unilateral approval, denial, and compromise impact

Recommended best practice for high-value or institutional deployments:

- Two or more independent evaluators
- Administrative and geographic diversity
- Explicit quorum thresholds defined in policy
- Independent signing keys per evaluator

This appendix is guidance only. Enforcement remains policy-defined.

---

## Appendix C — Operational Deployment Considerations (Informative)

Deployments SHOULD plan for:

- Mirror diversity across jurisdictions
- Monitoring of tick freshness and mirror agreement
- Runbooks for:
  - Epoch Clock divergence
  - Profile rotation
  - Evaluator key compromise
- Explicit operator education on fail-closed behavior

BPC prioritizes correctness and auditability over availability.

---

## 25. References (Informative)

* Epoch Clock v2.0.0 (pinned canonical profile_ref)
* RFC 2119, RFC 8949, RFC 8785
* BIP 174

---

## 26. Acknowledgements (Informative)

BPC builds on established primitives and norms from the Bitcoin and security engineering communities.

The author acknowledges:

- **Satoshi Nakamoto** and the Bitcoin contributor community, for the UTXO settlement model and the operational reality that high-assurance systems must remain consensus-compatible.
- The authors of **BIP 174 (PSBT)**, for standardizing interoperable transaction construction workflows that enable deterministic, tool-friendly signing pipelines.
- The authors and maintainers of **Taproot-related BIPs (BIP 340/341/342)**, for modernizing Bitcoin's script-path expressiveness and making execution-hardening patterns practical under current consensus.
- The **CBOR** and **JCS Canonical JSON** communities (RFC 8949 and RFC 8785), for canonical encoding foundations that make cross-implementation determinism and byte-stable hashing/signing achievable.
- The **post-quantum cryptography** research community and standardization efforts, for advancing practical signature schemes and verification discipline needed for long-lived security systems.

BPC is an independent specification and is not endorsed by any of the above parties.

---

## 27. Version History

**v1.0.0 (January 2026)**

- Initial public specification
- Refusal-first authorization model
- Deterministic precondition evaluation
- Canonical PSBTTemplateCanonical binding
- Unix-seconds (`_t`) time semantics aligned to EpochTick.t
- Explicit threat model and security invariants
- Bitcoin-native positioning and sovereignty guarantees

---

If you find this work useful and want to support it, you can do so here:  
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw
