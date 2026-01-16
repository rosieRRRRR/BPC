# Bitcoin Pre-Contracts (BPC)

**Deterministic Authorization Before Bitcoin Transaction Construction**

* **Specification Version:** 1.0.0
* **Status:** Public Beta - Draft for Review
* **Date:** January 2026
* **Author:** rosiea
* **Contact:** PQRosie@proton.me
* **Licence:** Apache License 2.0 - Copyright 2026 rosiea

---

## Abstract

Bitcoin Pre-Contracts (BPC) prevents the premature construction of Bitcoin transactions by requiring deterministic authorization before any spend artifact exists.

Traditional Bitcoin workflow constructs, signs, and broadcasts transactions immediately, assuming conditions were met. This creates a window where broadcast-valid transactions exist before authorization can be verified, enabling premature execution, replay attacks, and authorization bypass.

BPC inverts this model: authorization precedes construction. Wallets declare transaction intent, gather verifiable evidence, and evaluate preconditions deterministically. Only after explicit ALLOW does construction occur.

No Bitcoin consensus changes required.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

---

## 1. Why This Matters

### 1.1 The Problem: Premature Transaction Construction

Alice wants to sell 10 BTC to Bob for $1,000,000 USD via OTC settlement.

**Without BPC (Standard Bitcoin):**

```
T+0:    Alice constructs transaction: 10 BTC → Bob's address
T+0:    Alice signs transaction with her private key
        Transaction is now broadcast-valid
T+0:    Alice sends signed transaction to Bob for review
        ↓
T+30s:  Bob verifies transaction structure
T+60s:  Bob initiates bank wire for $1,000,000
        ↓
        [Bob expects Alice to wait for payment confirmation]
        ↓
T+120s: Alice broadcasts transaction immediately
        (ignores agreement to wait for payment)
        ↓
T+600s: Transaction confirms on Bitcoin blockchain
        Alice has 10 BTC worth of Bob's money
        Bob has no recourse
        ↓
Result: Alice executed before Bob's payment arrived
        Bob loses $1,000,000
```

**The vulnerability:** Once Alice constructs and signs the transaction, she controls when it broadcasts. Bob has no enforcement mechanism. The signed transaction is a loaded gun that Alice can fire at any time.

---

**With BPC:**

```
T+0:    Alice creates PreContractIntent
        Intent declares: "Send 10 BTC to Bob"
        Intent includes condition: "Requires proof of $1M payment"
        Intent is NOT a transaction (no signatures, not broadcast-valid)
        ↓
T+30s:  Bob reviews intent
        Bob initiates bank wire for $1,000,000
        ↓
T+3600s: Bob receives payment confirmation from bank
        Bank issues cryptographic proof of payment
        ↓
T+3700s: Bob submits EvidenceBundle
        Evidence includes: payment proof, current time, approvals
        ↓
T+3800s: BPC Evaluator processes evidence
        Evaluator checks:
        - Payment proof valid? YES
        - Amount correct? YES ($1M)
        - Recipient correct? YES (Alice)
        - Time conditions met? YES
        ↓
        Evaluator decision: ALLOW
        ↓
T+3900s: Alice's wallet receives ALLOW outcome
        ONLY NOW does wallet construct transaction
        Alice signs constructed transaction
        Alice broadcasts to Bitcoin network
        ↓
T+4500s: Transaction confirms on blockchain
        Both parties protected
        ↓
Result: Construction occurred ONLY AFTER payment verified
        No premature execution possible
        Bob protected
```

**The protection:** No broadcast-valid transaction exists until authorization succeeds. Alice cannot construct or sign until Bob's payment is verified. The authorization check is cryptographically bound to the transaction structure.

---

### 1.2 Attack Scenarios BPC Prevents

**Scenario 1: Premature Execution**

Without BPC: Signed transaction exists before conditions verified, can be broadcast early.

With BPC: No transaction exists until ALLOW. Construction gated by authorization.

---

**Scenario 2: Stale Authorization**

Without BPC: Transaction constructed days ago, broadcast after conditions no longer hold.

With BPC: Authorization includes expiry. Expired outcomes refuse execution.

---

**Scenario 3: Authorization Replay**

Without BPC: Single authorization used multiple times for different transactions.

With BPC: Authorization cryptographically bound to specific transaction structure via template hash. Single-use enforcement via decision_id tracking.

---

**Scenario 4: Evidence Manipulation**

Without BPC: No verifiable evidence requirements. Trust-based settlement.

With BPC: Evidence verified deterministically. Tampered evidence causes DENY. Time evidence anchored to Bitcoin-inscribed Epoch Clock profile.

---

### 1.3 Use Cases

BPC is optimized for transaction flows requiring deterministic authorization prior to construction.

BPC should be used when:

* Settlement occurs on Bitcoin L1
* Authorization can be determined before construction
* Conditions are expressible as verifiable evidence
* Failure to meet conditions MUST prevent construction
* An auditable authorization trail is required

**Typical applications include:**

* OTC settlement with delivery-vs-payment atomicity
* Policy-governed treasury operations (requires M-of-N approval)
* Time-bounded releases (vesting schedules, cliff unlocks)
* Multi-party approval workflows (corporate treasury, DAO operations)
* Proof-dependent settlement (oracle data, attestations)

**BPC should not be used when:**

* Transactions are simple single-signature spends (unnecessary overhead)
* Conditions require on-chain enforcement (use Bitcoin Script instead)
* Subjective judgment is required (BPC enforces deterministic verification only)
* Real-time execution without verification delay is required
* Privacy requirements outweigh auditability requirements

---

## 2. How BPC Works

### 2.1 Core Principle

BPC enforces a single, non-negotiable rule:

> **A broadcast-valid Bitcoin transaction MUST NOT be constructed until authorization succeeds.**

Instead of:
```
Construct → Sign → Broadcast → Hope conditions were met
```

BPC requires:
```
Declare intent → Gather evidence → Evaluate → Authorize → ONLY THEN construct
```

---

### 2.2 Protocol Flow (High-Level)

**Phase 1: Intent Declaration**

Wallet creates PreContractIntent declaring what transaction may be constructed.

Intent includes:
- Transaction structure (inputs, outputs, amounts)
- Required conditions (time bounds, approvals, proofs)
- Expiry window
- Contract identifier

Intent is NOT a transaction. It contains no signatures, no witness data, no broadcast-valid artifact.

---

**Phase 2: Evidence Gathering**

Required parties collect verifiable evidence:

- Time evidence (from Epoch Clock, anchored to Bitcoin)
- Approval signatures (from authorized parties)
- External proofs (payment confirmations, attestations)

Evidence is assembled into EvidenceBundle, cryptographically bound to intent.

---

**Phase 3: Deterministic Evaluation**

Evaluator receives Intent + Evidence and processes deterministically:

```
Input:  PreContractIntent + EvidenceBundle
Output: ALLOW | DENY | LOCKED
```

**Evaluation rules:**

- Time evidence valid and current?
- All required approvals present and valid?
- All required proofs present and verified?
- Intent not expired?
- All bindings match?

If ALL conditions satisfied: ALLOW
If ANY condition fails: DENY
If evaluation cannot complete safely: LOCKED

Same inputs MUST produce same output. No randomness, no discretion, no human judgment.

---

**Phase 4: Authorization Enforcement (Execution Gate)**

Wallet receives PreContractOutcome containing decision.

If decision is ALLOW:
- Wallet verifies outcome signature
- Wallet verifies all cryptographic bindings
- Wallet verifies outcome not expired
- Wallet verifies outcome not previously used
- Wallet constructs transaction
- Wallet presents transaction to signer
- Signer independently verifies outcome
- Signer signs transaction
- Wallet broadcasts to Bitcoin network

If decision is DENY or LOCKED:
- Wallet refuses construction
- No transaction artifact created
- Failure recorded in ExecutionResult
- User notified with reason

Any verification failure during execution gate refuses construction.

---

### 2.2A Time Evidence via Epoch Clock

BPC requires a deterministic, verifier-independent time source for all temporal authorization decisions, including expiry enforcement, replay prevention, and ordering guarantees.

BPC uses **Epoch Clock** as its sole source of time evidence.

Epoch Clock provides a Bitcoin-anchored time authority with the following properties:

**Time Authority Properties:**

* A canonical Epoch Clock profile inscribed on Bitcoin at ordinal
  `439d7ab1972803dd984bf7d5f05af6d9f369cf52197440e6dda1d9a2ef59b6ebi0`
* Deterministic tick derivation anchored to Bitcoin blockchain state
* Cryptographically signed ticks verifiable against the canonical profile
* Verifier independence: any conformant implementation derives identical ticks from the same blockchain state

Epoch Clock introduces no execution authority and no custody role. It supplies verifiable time only.

---

#### How Time Evidence Is Used

Epoch Clock time evidence is incorporated into BPC as follows:

1. **Profile Resolution**
   The canonical Epoch Clock profile is resolved from its Bitcoin inscription.

2. **Tick Derivation**
   The current tick is derived deterministically from Bitcoin blockchain state, as defined by the Epoch Clock specification.

3. **Signature Verification**
   The tick signature is verified against the public key defined in the canonical profile.

4. **Evidence Binding**
   The verified tick is included in the `EvidenceBundle` and bound to the evaluated intent.

All evaluators and signers MUST verify time evidence independently.

---

#### Availability and Fallback Rules

* Epoch Clock time is available whenever Bitcoin blockchain state is available.
* Implementations MAY obtain ticks from decentralized mirrors for transport convenience.
* Implementations MUST verify all ticks against the canonical profile.
* Implementations MUST NOT fall back to system time, wall-clock time, or platform time sources.

If valid time evidence cannot be obtained or verified, evaluation MUST emit `LOCKED`.

---

#### Rationale

BPC authorization depends on deterministic time semantics.

Allowing system time or discretionary fallbacks would:

* break determinism,
* enable replay ambiguity,
* introduce verifier disagreement,
* undermine auditability.

Epoch Clock ensures that all temporal decisions in BPC are derived from a common, Bitcoin-anchored reference without introducing new execution authority.

For the full Epoch Clock specification, see: https://github.com/rosieRRRRR/epoch-clock

---

### 2.3 Visual Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                     BPC AUTHORIZATION FLOW                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  [USER INITIATES]                                                │
│         │                                                         │
│         ▼                                                         │
│  ┌─────────────────┐                                             │
│  │ PreContractIntent│  ← Declares what may be constructed        │
│  │ (No signatures) │                                             │
│  └────────┬─────────┘                                             │
│           │                                                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │ EvidenceBundle  │  ← Verifiable inputs                        │
│  │ - Time evidence │                                             │
│  │ - Approvals     │                                             │
│  │ - Proofs        │                                             │
│  └────────┬─────────┘                                             │
│           │                                                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │   EVALUATOR     │  ← Deterministic processing                 │
│  │                 │                                             │
│  │  Same inputs    │                                             │
│  │  Same output    │                                             │
│  └────────┬─────────┘                                             │
│           │                                                       │
│      ┌────┴────┬───────────┐                                     │
│      ▼         ▼           ▼                                     │
│   [ALLOW]   [DENY]    [LOCKED]                                   │
│      │         │           │                                     │
│      │         └───────────┴────> REFUSE (No tx created)         │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────┐                                             │
│  │ EXECUTION GATE  │  ← Verify outcome + bindings               │
│  │                 │                                             │
│  │ - Check sig     │                                             │
│  │ - Check expiry  │                                             │
│  │ - Check replay  │                                             │
│  │ - Check bindings│                                             │
│  └────────┬─────────┘                                             │
│           │                                                       │
│      All valid?                                                  │
│           │                                                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │ CONSTRUCT TX    │  ← ONLY NOW does transaction exist         │
│  └────────┬─────────┘                                             │
│           │                                                       │
│           ▼                                                       │
│  ┌─────────────────┐                                             │
│  │  SIGN & BROADCAST│ ← Standard Bitcoin from here              │
│  └─────────────────┘                                             │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Trust Model and Evaluator Authority

### 3.1 The Core Question

BPC introduces external evaluation before transaction construction. This raises an immediate question:

**"Who runs the evaluator, and why should I trust it?"**

---

### 3.2 What Evaluators Do

Evaluators perform deterministic verification:

- Check that evidence is valid (signatures verify, proofs are well-formed)
- Check that time evidence is current (Epoch Clock verification)
- Check that all required conditions are present
- Output ALLOW, DENY, or LOCKED based on deterministic rules

**Evaluators do NOT:**

- Hold funds or keys
- Have custody or control over Bitcoin
- Make subjective judgments
- Have discretion in decision-making
- Compel or prevent signing
- Execute or broadcast transactions

---

### 3.3 Comparison to Miner Trust Assumptions

Bitcoin already relies on miners for transaction inclusion. Miners have significant power:

- Can refuse to include any transaction
- Can reorder transactions within blocks
- Can front-run transaction content
- Have temporary exclusive visibility during private relay (Seal-360)

**Yet Bitcoin considers this acceptable because:**

1. Miners cannot steal funds (don't control keys)
2. Miners cannot prevent execution permanently (fallback to other miners)
3. Miner behavior is economically rational (fees incentivize inclusion)
4. Miner power is temporary (only during block construction)

**BPC evaluators have LESS power than miners:**

1. Evaluators cannot steal funds (don't control keys)
2. Evaluators cannot prevent execution permanently (user can bypass BPC)
3. Evaluator behavior is deterministic (no economic discretion)
4. Evaluator power is temporary (only during authorization phase)

**Additional protections:**

- Evaluator logic is open-source and auditable
- Multiple evaluators can be used (quorum)
- Evaluation is deterministic (same inputs = same output)
- Signers verify outcomes independently (no blind trust)
- Users retain sovereignty (can refuse to use BPC)

---

### 3.4 Sovereignty Preservation

BPC preserves Bitcoin's core principle: the signer is sovereign.

**Evaluators cannot:**

- Compel signing
- Infer consent
- Proceed on behalf of key holder
- Override signer refusal

**Signers retain final control:**

- Signer receives outcome for independent verification
- Signer checks signature on outcome
- Signer checks all binding fields match
- Signer checks outcome not expired or replayed
- Signer approves final transaction before signing
- Signer can refuse at any point

**If evaluator misbehaves:**

- Signer detects mismatch and refuses
- No transaction is constructed
- No funds are at risk
- User can bypass BPC entirely for future transactions

---

### 3.5 Comparison to Trusted Third Parties

**Traditional trusted third party:**

- Holds funds or keys
- Has custody and control
- Can unilaterally execute transactions
- Single point of failure
- Breach results in fund loss

**BPC Evaluator:**

- Holds no funds or keys
- Has no custody or control
- Cannot execute transactions
- Can be redundant (multiple evaluators)
- Breach results in denial, not theft

**Key distinction:** BPC evaluators verify evidence, they do not control assets.

---

### 3.6 Time Authority (Epoch Clock)

BPC requires verifiable time for:

- Expiry enforcement
- Replay prevention
- Temporal ordering

**Time source:** Epoch Clock v2, anchored to Bitcoin via inscription.

**Epoch Clock properties:**

- Profile inscribed on Bitcoin blockchain (immutable reference)
- Tick progression cryptographically signed
- Multiple mirrors provide redundancy
- Agreement protocol requires mirror consensus
- No single point of failure

**If Epoch Clock unavailable:**

- Evaluation emits LOCKED
- Execution refuses
- No fallback to system time
- Fail-closed behavior preserved

**Rationale:**

Time authority already exists in Bitcoin:

- Block timestamps establish temporal ordering
- Miners control time within consensus rules
- Locktimes depend on block height or timestamps

BPC uses Bitcoin-anchored time via inscription, maintaining consistency with Bitcoin's existing time model.

---

### 3.7 Multi-Evaluator Quorum (Optional)

For increased assurance, deployments MAY use multiple evaluators:

**Quorum model:**

- M-of-N evaluators must emit ALLOW
- All ALLOW outcomes must have identical bindings
- Any DENY causes overall DENY
- Any LOCKED causes overall LOCKED

**Benefits:**

- Tolerates individual evaluator failure
- Reduces single evaluator trust
- Increases availability (if some evaluators down)

**Trade-offs:**

- Increased complexity
- Higher latency
- Coordination overhead

Quorum is OPTIONAL. Single evaluator is conformant.

---

### 3.8 Trust Summary

BPC introduces authorization verification, not asset custody.

Evaluators verify evidence deterministically using the same rules available to all participants.

Signers independently verify all outcomes before signing.

Users retain sovereignty and can bypass BPC if evaluator behavior is unacceptable.

This trust model is comparable to Bitcoin's existing reliance on miners for transaction inclusion.

---

### 3.9 Design Rationale and Common Objections (Non-Normative)

This section addresses predictable review objections that arise from misunderstanding Bitcoin Pre-Contracts’ authority model, evaluator role, and execution semantics.
Nothing in this section introduces new requirements or modifies any normative rule in this specification.

---

#### Does BPC add trust in evaluators?

No.

Evaluators in BPC do not gain custody, execution authority, or signing capability.

* Evaluators cannot construct transactions.
* Evaluators cannot sign transactions.
* Evaluators cannot broadcast transactions.
* Evaluators cannot compel execution.

Evaluators verify evidence and emit an authorization outcome. Execution remains under signer control and is enforced locally by the wallet and signer.

If an evaluator misbehaves, produces an invalid outcome, or becomes unavailable, execution is refused. No funds are at risk.

---

#### Does BPC centralise transaction control?

No.

BPC preserves signer sovereignty and Bitcoin’s censorship resistance.

* BPC is optional and may be bypassed entirely by the user.
* Authorization does not guarantee execution.
* Execution requires independent signer verification at signing time.

No evaluator can permanently block execution, because users retain the ability to construct and broadcast standard Bitcoin transactions without BPC.

BPC governs whether a transaction may exist, not whether Bitcoin will accept it.

---

#### Why is authorization enforced before construction?

Because construction itself creates risk.

Once a broadcast-valid transaction exists, control over execution timing is lost. BPC eliminates this risk by enforcing authorization before any spend artifact exists.

On-chain mechanisms cannot prevent premature construction, cached signatures, or deferred broadcast. This invariant must be enforced in the wallet and signer execution path.

BPC reorders authority without changing Bitcoin consensus.

---

#### Why is this not implemented in Script or consensus?

Because BPC does not govern validity. It governs existence.

* Script enforces spending conditions after construction.
* Consensus validates transactions after broadcast.
* Neither can prevent a signed transaction from existing prematurely.

BPC operates entirely off-chain to control execution timing and authority boundaries without introducing global consensus risk.

---

#### Isn’t this too complex for wallets?

Security systems are evaluated by how they constrain failure.

BPC confines complexity to a small number of well-defined components:

* intent declaration,
* deterministic evaluation,
* execution gating,
* signer verification.

The alternative is implicit complexity spread across UI conventions, trust assumptions, and informal agreements.

BPC replaces informal trust with explicit, auditable control.

---

#### What if the evaluator is unavailable or malicious?

BPC fails closed.

* Missing or unverifiable evidence results in denial or lock.
* Expired or replayed outcomes are refused.
* Invalid signatures are rejected by the signer.

No transaction is constructed in these cases. No funds are exposed.

Evaluator failure prevents execution. It does not create execution risk.

---

#### Does BPC weaken Bitcoin’s trust model?

No.

BPC does not introduce:

* custody delegation,
* execution delegation,
* inclusion guarantees,
* new consensus assumptions.

It preserves Bitcoin’s core principle: only the signer authorises spending.

BPC enforces when authority may be exercised, not who holds it.

---

#### Is BPC required for all transactions?

No.

BPC is intended for transactions where premature execution is unacceptable, including:

* high-value settlement,
* policy-governed treasury operations,
* time-bounded releases,
* multi-party approval flows.

Simple transactions remain better served by standard Bitcoin workflows. 

---

## 4. Core Protocol

### 4.1 Core Security Invariant (Normative)

The following invariant MUST hold for any conformant implementation:

> **No broadcast-valid spend artifact may exist before authorization succeeds.**

This includes:

- Raw transactions
- Partially signed PSBTs with sufficient signatures
- Recoverable signing material
- Any artifact that could be finalized without re-evaluation

Violation of this invariant is critical non-conformance.

---

### 4.2 PreContractIntent

Declares what transaction may be constructed.

**Properties:**

- Non-authoritative (not a transaction)
- Safe to publish (contains no signing material)
- Content-addressed (intent_hash binds structure)
- Time-bounded (expiry_t enforces validity window)

**Structure:**

```
PreContractIntent = {
  version: 1,
  contract_id: bstr,              // Unique contract identifier
  expiry_t: uint,                 // Epoch Clock tick (expiry)
  session_id: bstr,               // Unique session identifier
  intent_body: {
    psbt_template: PSBTTemplateCanonical,
    conditions: {
      required_approvals: [* ApprovalRequirement] / null,
      required_proofs: [* ProofRequirement] / null,
      time_constraints: TimeConstraints / null
    }
  },
  intent_hash: bstr(32)           // SHA-256(DetCBOR(intent without hash))
}
```

**Canonical hash computation:**

```
canonical_bytes = DetCBOR(intent excluding intent_hash field)
intent_hash = SHA-256(canonical_bytes)
```

**Validation rules:**

- version MUST be 1
- contract_id MUST be unique per contract
- expiry_t MUST be Epoch Clock tick (not Unix timestamp)
- session_id MUST be unique per evaluation attempt
- intent_hash MUST match computed hash
- psbt_template MUST be PSBTTemplateCanonical (Section 4.3)
- conditions MUST be verifiable deterministically
- Intent MUST NOT contain signatures, witnesses, or signing material

---

### 4.3 PSBTTemplateCanonical

Defines authorized transaction structure without signatures.

**Structure:**

```
PSBTTemplateCanonical = {
  version: 1,
  inputs: [* {
    txid: bstr(32),
    vout: uint,
    sequence: uint
  }],
  outputs: [* {
    script_pubkey: bstr,
    amount_sats: uint
  }],
  locktime: uint,
  fee_rate_sat_vb: uint / null    // null = not binding
}
```

**Ordering requirements:**

- Inputs MUST be sorted lexicographically by (txid, vout)
- Outputs MUST be sorted lexicographically by (amount_sats, script_pubkey)

**Exclusion requirements:**

Template MUST NOT include:

- Signatures
- Witnesses
- Final scripts
- UTXO data
- Key material or secrets
- Unknown or proprietary PSBT fields

**Replace-By-Fee semantics:**

RBF determined exclusively by input sequence values:

- sequence < 0xFFFFFFFE → replaceable
- sequence >= 0xFFFFFFFE → non-replaceable

No independent flags permitted.

**Fee binding:**

- If fee_rate_sat_vb is uint → binding (final transaction MUST match rate within tolerance)
- If fee_rate_sat_vb is null → not binding (any fee acceptable)

**Template hash computation:**

```
canonical_bytes = DetCBOR(psbt_template)
template_hash = SHA-256(canonical_bytes)
```

This hash cryptographically binds authorization to transaction structure.

---

### 4.4 EvidenceBundle

Supplies verifiable inputs required for authorization.

**Structure:**

```
EvidenceBundle = {
  version: 1,
  contract_id: bstr,
  intent_hash: bstr(32),
  time_evidence: {
    tick_bytes: bstr,             // JCS-canonical JSON from Epoch Clock
    profile_ref: tstr,            // Must match pinned profile
    validated_at_mono_ms: uint    // Local monotonic time at validation
  },
  approvals: [* Approval] / null,
  proofs: {* tstr => any} / null
}

```

**Time evidence (REQUIRED):**

All BPC evaluations MUST include valid time evidence.

**Time evidence validation:**

1. profile_ref MUST match pinned profile: `ordinal:439d7ab1972803dd984bf7d5f05af6d9f369cf52197440e6dda1d9a2ef59b6ebi0`
2. Signature MUST verify against profile's public key.
3. tick_bytes MUST be byte-identical JCS-canonical JSON.
4. `validated_at_mono_ms` is used solely as an ingress freshness guard to prevent replay of stale EvidenceBundles. This field MUST NOT be referenced during deterministic evaluation logic and MUST NOT influence the evaluation outcome.
5. Tick reuse permitted only while: `mono_now_ms - validated_at_mono_ms <= 900000` (15 minutes).
6. If time unavailable or stale, evaluation MUST emit LOCKED.

---

### 4.5 PreContractOutcome

Signed authorization result.

**Structure:**

```
PreContractOutcome = {
  version: 1,
  decision_id: bstr,              // Unique per decision (replay prevention)
  decision: "ALLOW" | "DENY" | "LOCKED",
  bound_intent_hash: bstr(32),
  bound_contract_id: bstr,
  bound_session_id: bstr,
  bound_psbt_hash: bstr(32),
  issued_t: uint,                 // Epoch Clock tick when issued
  valid_until_t: uint,            // Epoch Clock tick expiry
  denial_reason: tstr / null,     // Present if DENY or LOCKED
  sig: bstr,
  sig_alg: tstr
}
```

**Decision semantics:**

- ALLOW: All conditions satisfied, construction authorized
- DENY: One or more conditions failed, construction refused
- LOCKED: Evaluation could not complete safely, construction refused

#### Decision Semantics Summary

| Decision | Meaning | When Emitted |
|--------|--------|-------------|
| **ALLOW** | All conditions satisfied; construction authorized | Successful evaluation |
| **DENY** | One or more conditions failed | Policy, proof, or approval failure |
| **LOCKED** | Evaluation could not complete safely | Time ambiguity, mirror divergence, system uncertainty |

**Binding fields:**

Outcome cryptographically binds to:

- bound_intent_hash: Specific intent evaluated
- bound_contract_id: Specific contract
- bound_session_id: Specific evaluation session
- bound_psbt_hash: Specific transaction structure

Any mismatch between outcome bindings and actual construction MUST refuse.

**Single-use enforcement:**

Each decision_id MAY be used exactly once for execution.

Executors MUST maintain durable replay guard tracking used decision_ids.

Attempted reuse MUST refuse with E_OUTCOME_REPLAYED.

**Expiry enforcement:**

Outcome valid only while:

```
current_t <= valid_until_t
```

Expired outcomes MUST refuse with E_OUTCOME_EXPIRED.

**Signature verification:**

Outcome MUST be signed by evaluator.

Executors MUST verify signature before accepting outcome.

Signature covers canonical DetCBOR encoding of outcome excluding sig field.

---

### 4.6 Deterministic Evaluation (Normative)

Evaluators MUST satisfy:

> **Same Intent + Same Evidence + Same Verified Time → Same Outcome**

**Determinism requirements:**

1. All evaluation logic MUST be deterministic.
2. Deterministic evaluation MUST NOT reference wall-clock time, monotonic time, randomness, or external state.
3. All time-based admissibility checks (including monotonic freshness validation) occur strictly prior to evaluation as ingress guards and are not part of the deterministic evaluation function.
4. Same inputs MUST produce byte-identical output.
5. Cross-platform equivalence REQUIRED.
6. Canonical encoding stability REQUIRED.

**Evaluation process:**

```
1. Validate Intent structure and hash
2. Validate Evidence structure and bindings
3. Validate time evidence (Epoch Clock)
4. Extract current_t from time evidence
5. Check intent not expired: current_t <= intent.expiry_t
6. Validate all required approvals
7. Validate all required proofs
8. Evaluate conditions deterministically
9. If all conditions satisfied: ALLOW
10. If any condition failed: DENY
11. If evaluation cannot complete: LOCKED
12. Sign outcome with evaluator key
13. Return PreContractOutcome

```

**Missing, stale, malformed, or unverifiable inputs MUST result in DENY or LOCKED.**

---

### 4.7 Execution Gate (Normative)

Construction and signing MUST occur only if ALL of the following hold:

**Outcome validation:**

1. decision == "ALLOW"
2. Outcome signature valid
3. current_t <= valid_until_t (not expired)
4. decision_id not previously used (not replayed)

**Binding validation:**

5. bound_intent_hash matches actual intent hash
6. bound_contract_id matches actual contract_id
7. bound_session_id matches actual session_id
8. bound_psbt_hash matches actual template hash

**Transaction validation:**

9. Constructed transaction matches PSBTTemplateCanonical exactly:
* Input count, ordering, txid, vout, sequence
* Output count, ordering, script_pubkey, amount_sats
* Locktime value
* RBF semantics (via sequence)


Permitted differences:
* Witness stack contents (signatures)
* scriptSig content (signatures)
* Signature-length variance


10. If `fee_rate_sat_vb` was binding, the final transaction fee rate MUST match the template within an explicit tolerance. Unless otherwise specified in the `PreContractIntent`, the default tolerance is ±1 sat/vB. An implementation MAY allow `fee_rate_tolerance_sat_vb` to be explicitly declared in the `PreContractIntent`; if declared, that value overrides the default tolerance.

**Any validation failure MUST refuse construction.**

---

### 4.8 Signer Verification (Normative)

Signers MUST independently verify outcomes before signing.

Host software is untrusted. Signers MUST NOT rely on host verification.

**Required signer checks:**

1. Verify outcome signature
2. Verify decision == "ALLOW"
3. Verify current_t <= valid_until_t
4. Verify bound_intent_hash matches displayed intent
5. Verify bound_psbt_hash matches displayed template
6. Verify template matches final transaction structure
7. Verify decision_id not previously signed

**Hardware wallet implementations:**

Hardware wallets MUST display:
- Contract ID
- Transaction structure (inputs, outputs, amounts)
- Authorization expiry
- Evaluator identity

Hardware wallets MUST refuse to sign if any verification fails.

Any bypass of signer verification invalidates BPC conformance claims.

---

## 5. Relationship to Seal-360 and Execution Boundaries

BPC and Seal-360 are **orthogonal, composable protocols**.
They MAY be used together, but neither depends on the other for correctness.

This section defines:

* protocol boundaries,
* the normative definition of a *spend artifact*,
* canonical ordering rules,
* fee-binding semantics,
* and failure-domain separation.

Nothing in this section alters earlier authorization semantics.
It defines **where authority is allowed to exist**.

---

### 5.1 Protocol Boundaries

**BPC (this specification):**

* Governs authorization **before** transaction construction
* Determines **whether** a transaction may exist
* Enforces preconditions via deterministic evaluation
* Outputs: `ALLOW`, `DENY`, or `LOCKED`

**Seal-360 (separate specification):**

* Governs execution **after** a transaction is signed
* Determines **how** to avoid public mempool exposure
* Uses encrypted direct-to-miner submission with mandatory public fallback
* Inputs: a signed transaction (from BPC or any standard wallet)

Seal-360 is **out of scope** for BPC conformance.
Failure or absence of Seal-360 MUST NOT affect BPC authorization correctness.

---

### 5.2 Normative Definition: Spend Artifact

A **spend artifact** is any digital object from which a broadcast-valid Bitcoin
transaction can be produced **without re-running BPC authorization evaluation**.

No spend artifact MAY exist prior to receipt and verification of a valid
`PreContractOutcome` with `decision == ALLOW`.

This definition is **normative** and applies to all BPC-conformant implementations.

---

#### 5.2.1 Prohibited Artifacts (MUST NOT Exist Before `ALLOW`)

The following objects are **spend artifacts** and MUST NOT exist, be cached,
or be reconstructable before a valid `ALLOW` outcome is verified:

| Artifact Type                 | Description                                              |
| ----------------------------- | -------------------------------------------------------- |
| Fully signed transaction      | Raw Bitcoin transaction ready for broadcast              |
| Complete PSBT                 | PSBT containing all required final scripts or witnesses  |
| Cached signatures             | Any stored ECDSA/Schnorr signatures usable for assembly  |
| Auto-sign configuration       | Any policy that signs without re-verifying authorization |
| Pre-signed or deferred spends | Transactions held for later broadcast                    |

Existence of any prohibited artifact prior to `ALLOW` is a **critical
conformance violation**.

---

#### 5.2.2 Permitted Objects (NOT Spend Artifacts)

The following MAY exist prior to authorization and do not violate this specification:

| Object                                      | Rationale                        |
| ------------------------------------------- | -------------------------------- |
| `PreContractIntent`                         | Not a Bitcoin transaction        |
| Unsigned PSBT                               | Cannot be broadcast or finalized |
| Partially signed PSBT (insufficient quorum) | Cannot satisfy spending rules    |
| Public keys                                 | Cannot authorize spending        |
| Script or template data                     | Contains no signing material     |
| `EvidenceBundle`                            | Authorization input only         |

---

#### 5.2.3 Implementation Requirements by Role

**Wallet Implementations MUST:**

* NOT generate or cache signatures before verifying `ALLOW`
* NOT pre-construct signed transactions “optimistically”
* Ensure no complete PSBT exists in storage prior to authorization

**External Evaluator Implementations MUST:**

* NOT generate or return signatures
* NOT persist signature material across evaluation sessions
* MAY return unsigned templates for inspection only

**Signer / Hardware Wallet Implementations MUST:**

* Independently verify `PreContractOutcome`
* Refuse signing if `ALLOW` is absent, expired, or mismatched
* NOT support auto-sign behaviour for BPC flows
* Display intent and authorization context to the user

**Multi-Signature Implementations MUST:**

* Require all cosigners to independently verify `ALLOW`
* Prohibit partial signature collection before authorization
* Perform signature aggregation only after verification

---

#### 5.2.4 Audit Requirement

Conformant implementations MUST be able to demonstrate, under audit, that:

* No spend artifact existed before `ALLOW`
* All signatures were generated after outcome verification
* Any failure before `ALLOW` resulted in zero signing material

This MAY be satisfied by deterministic logs, execution traces,
or signer-side verification evidence.

---

### 5.3 Canonical Input and Output Ordering (Normative)

Canonical ordering ensures deterministic hashing, cross-implementation
consistency, and unambiguous authorization binding.

These rules apply to all `PSBTTemplateCanonical` objects.

---

#### 5.3.1 Input Ordering

Inputs MUST be ordered using **BIP-69** lexicographical ordering:

1. `txid` (32-byte hash, raw byte order, ascending)
2. `vout` (32-bit unsigned integer, ascending)

This ordering MUST be applied prior to template hashing.

---

#### 5.3.2 Output Ordering (BPC-Specific)

Outputs MUST be ordered using the following total order:

1. `amount_sats` (ascending, smallest first)
2. `script_pubkey` (lexicographical byte order)

This ordering is **intentionally distinct from BIP-69**.

---

#### 5.3.3 Rationale

BPC uses **amount-first ordering** to:

* Provide deterministic disambiguation for authorization-bound templates
* Simplify audit reasoning for treasury and high-value settlement
* Ensure a stable, total ordering suitable for template hash binding

Wallets implementing both BPC and non-BPC transactions MUST clearly
distinguish which ordering applies.

---

#### 5.3.4 Compatibility Note

BPC output ordering MUST NOT be described as BIP-69 compliant.

Any implementation claiming BPC conformance MUST apply the ordering
defined in this section exactly.

---

#### 5.3.5 Test Vector (Normative)

Given the outputs:

```
Output A: 50000 sats, scriptPubKey = 0x0014abcd...
Output B: 50000 sats, scriptPubKey = 0x00140123...
Output C: 100000 sats, scriptPubKey = 0x0014ffff...
```

Canonical order:

1. Output B
2. Output A
3. Output C

---

### 5.4 Fee Binding Tolerance (Normative)

When `fee_rate_sat_vb` is binding in `PSBTTemplateCanonical`,
the executor MUST enforce fee conformity deterministically.

Unless otherwise specified in `PreContractIntent`, the default tolerance is:

```
±1 sat/vB
```

If a different tolerance is required, it MUST be explicitly encoded in
`PreContractIntent` and included in the intent hash.

Undefined or implicit tolerance is non-conformant.

---

### 5.5 Failure Domain Separation (Normative)

BPC separates **authorization failure** from **execution failure**.

---

#### 5.5.1 Authorization Failure (Evaluator Domain)

Occurs when the evaluator emits `DENY` or `LOCKED`.

Effects:

* No transaction construction permitted
* No signing permitted
* No spend artifact may exist
* Failure MUST be recorded
* Fail-closed semantics apply

---

#### 5.5.2 Execution Failure (Wallet / Signer Domain)

Occurs **after** a valid `ALLOW`, but before or during execution.

Examples:

* Outcome expiry
* Binding mismatch
* Replay detection
* Signer refusal
* Broadcast or Seal-360 failure

Effects:

* Execution attempt terminates
* Authorization itself is not retroactively invalidated
* New execution requires a fresh attempt

---

#### 5.5.3 Error Classification

| Error Class       | Origin          | Effect             |
| ----------------- | --------------- | ------------------ |
| Evaluation Errors | Evaluator       | `DENY` or `LOCKED` |
| Gate Errors       | Wallet / Signer | `REFUSE`           |
| Broadcast Errors  | Network         | Retry or fallback  |

Implementations MUST NOT conflate evaluator decisions with execution refusal.

---

### 5.6 Summary of Section 5

Section 5 defines BPC’s **security boundary**:

* Authorization precedes construction
* Spend artifacts MUST NOT exist before `ALLOW`
* Ordering and fee rules are deterministic and explicit
* Authorization and execution failures are strictly separated
* Seal-360 is optional and orthogonal

Any implementation violating these constraints is **non-conformant** with BPC.

---

### 5.7 BPC and Seal-360 Composition (Informative)

```
┌─────────────────────────────────────────────────┐
│        BPC + Seal-360 COMPOSITION               │
├─────────────────────────────────────────────────┤
│                                                 │
│  [BPC SCOPE: Authorization Before Construction] │
│         │                                       │
│         ▼                                       │
│  Does transaction meet                          │
│  preconditions?                                 │
│         │                                       │
│    ┌────┴────┐                                  │
│    │ ALLOW?  │                                  │
│    └────┬────┘                                  │
│         │                                       │
│         ▼                                       │
│  Construct + Sign                               │
│  (Standard Bitcoin)                             │
│         │                                       │
│         ▼                                       │
│  [Seal-360 SCOPE: Execution After Signing]      │
│         │                                       │
│         ▼                                       │
│  Encrypted submission                           │
│  to miner                                       │
│         │                                       │
│         ▼                                       │
│  Public fallback                                │
│  (if private relay fails)                       │
│                                                 │
│  BPC answers: “May this exist?”                 │
│  Seal-360 answers: “How is this delivered?”    │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## 6. Error Handling and Codes

### 6.1 Error Categories

BPC defines four error categories:

* **Time errors**: Epoch Clock failures.
* **Binding errors**: Hash or ID mismatches.
* **Evaluation errors**: Condition failures.
* **Execution errors**: Construction or signing failures.

#### 6.1.1 Evaluation vs Execution Errors

Errors are classified by the stage at which they occur:

* **Evaluation Errors** result in a deterministic `DENY` or `LOCKED` outcome produced by the Evaluator.
* **Execution Gate Errors** result in a `REFUSE` action by the Executor or Wallet, even if evaluation previously returned `ALLOW`.

An error code MAY appear in both contexts, but its effect depends on the stage at which it is encountered. For example, `E_OUTCOME_EXPIRED` represents an evaluation outcome expiration when detected by the Evaluator, and a refusal condition when detected by the Execution Gate.

### 6.2 Time Errors

**E_TICK_MISSING**
- Cause: No time evidence in EvidenceBundle
- Action: Emit LOCKED

**E_TICK_PROFILE_MISMATCH**
- Cause: profile_ref does not match pinned profile
- Action: Emit LOCKED

**E_TICK_SIG_INVALID**
- Cause: Tick signature verification failed
- Action: Emit LOCKED

**E_MIRROR_DIVERGENCE**
- Cause: Mirrors returned different tick values
- Action: Emit LOCKED

**E_TICK_STALE**
- Cause: Tick reuse window exceeded (>15 minutes)
- Action: Emit LOCKED

### 6.3 Binding Errors

**E_INTENT_HASH_MISMATCH**
- Cause: Computed intent_hash does not match provided hash
- Action: Refuse evaluation

**E_INTENT_EXPIRED**
- Cause: current_t > intent.expiry_t
- Action: Emit DENY

**E_OUTCOME_EXPIRED**
- Cause: current_t > outcome.valid_until_t
- Action: Refuse execution

**E_OUTCOME_REPLAYED**
- Cause: decision_id already used
- Action: Refuse execution

**E_PSBT_TEMPLATE_MISMATCH**
- Cause: bound_psbt_hash does not match actual template
- Action: Refuse execution

**E_PSBT_TEMPLATE_NONCANONICAL**
- Cause: Template contains excluded fields or wrong ordering
- Action: Refuse evaluation

### 6.4 Evaluation Errors

**E_POLICY_DENY**
- Cause: Evaluation policy explicitly denied authorization
- Action: Emit DENY

**E_APPROVALS_MISSING**
- Cause: Required approvals not present in evidence
- Action: Emit DENY

**E_APPROVAL_INVALID**
- Cause: Approval signature invalid or fields incorrect
- Action: Emit DENY

**E_PROOF_INVALID**
- Cause: Required proof missing or verification failed
- Action: Emit DENY

### 6.5 Execution Errors

**E_SIGNING_FAILED**
- Cause: Signer refused or signing operation failed
- Action: Refuse execution

**E_BROADCAST_FAILED**
- Cause: Transaction broadcast to network failed
- Action: Mark execution as FAILED (transaction exists but not sent)

**E_FINAL_TX_TEMPLATE_DIVERGENCE**
- Cause: Final transaction does not match PSBTTemplateCanonical
- Action: Refuse execution

---

## 7. Conformance Requirements

### 7.1 Evaluator Conformance

A conformant evaluator implementation MUST:

1. Use DetCBOR for all BPC-native objects.
2. Integrate Epoch Clock v2 with pinned profile.
3. Implement deterministic evaluation where the same inputs MUST produce the same outputs without referencing wall-clock time, monotonic time, or randomness.
4. Enforce PSBTTemplateCanonical validation, including BIP-69 lexicographical ordering.
5. Emit only ALLOW, DENY, or LOCKED decisions.
6. Sign all outcomes with evaluator key.
7. Enforce single-use decision_ids.
8. Implement all time evidence validation rules, treating `validated_at_mono_ms` strictly as an ingress guard.
9. Implement all approval validation rules.
10. Refuse evaluation on any ambiguity or missing input.

---

### 7.2 Executor Conformance

A conformant executor implementation MUST:

1. Verify the `PreContractOutcome` signature using the trusted evaluator's public key before proceeding to transaction construction.
2. Enforce the `valid_until_t` constraint by comparing it against the current time provided by a validated Epoch Clock tick.
3. Maintain a local registry of used `decision_id` values to prevent outcome replay.
4. Strictly validate that the `intent_hash`, `contract_id`, and `psbt_hash` in the outcome match the local transaction context.
5. Construct the final Bitcoin transaction to match the `PSBTTemplateCanonical` exactly, ensuring no deviation in input/output count, sequence, or ordering.
6. Apply canonical ordering as defined in PSBTTemplateCanonical:
   - Inputs MUST follow BIP-69 ordering.
   - Outputs MUST follow BPC-specific ordering (amount_sats, script_pubkey).
7. Verify that the final transaction fee rate matches the `fee_rate_sat_vb` within the default tolerance of ±1 sat/vB, or the specific `fee_rate_tolerance_sat_vb` if declared in the intent.
8. Refuse to sign or broadcast the transaction if any validation check in the Execution Gate fails.
9. Handle `E_OUTCOME_EXPIRED` as a terminal refusal condition if detected at the Execution Gate.
10. Ensure that RBF signaling in the final transaction is derived exclusively from the template sequence values.

### 7.3 Signer Conformance

A conformant signer (hardware wallet, HSM) implementation MUST:

1. Independently verify the `PreContractOutcome` signature using the trusted evaluator's public key.
2. Independently verify that all binding fields (`intent_hash`, `contract_id`, `psbt_hash`) in the outcome match the transaction being presented for signing.
3. Independently verify that the outcome is not expired by checking the `valid_until_t` against a verified Epoch Clock tick.
4. Display the Contract ID, the transaction structure (inputs, outputs, and amounts), the authorization expiry, and the Evaluator identity to the user for manual review.
5. Maintain a local record of `decision_id` values to prevent signing multiple transactions for a single authorization.
6. Refuse to sign the transaction if any of the above verification steps fail.
7. Ensure that it does not rely on host software for any part of the verification process to maintain the protocol's security invariants.

---

### 7.4 Interoperability

Conformant implementations MUST interoperate with:
- Any conformant evaluator
- Any conformant executor
- Any conformant signer

Cross-implementation incompatibility is a conformance violation.

---

## 8. Deployment Guidance

### 8.1 Implementation Classes

**Class 1: Wallet-Only Enforcement**

Local evaluation within wallet, no external evaluator.

Suitable for:
- Personal custody
- Simple time-based conditions
- Single-party authorization

Not suitable for:
- Multi-party workflows
- External proof verification
- High-assurance settlement

**Class 2: External Evaluator + Wallet**

Evaluator performs evaluation, wallet verifies outcome.

Suitable for:
- OTC settlement
- Treasury operations
- Policy enforcement

Requires:
- Trusted evaluator deployment
- Secure evaluator key management
- Wallet integration with evaluator API

**Class 3: Multi-Evaluator Quorum**

Requires M-of-N evaluator agreement.

Suitable for:
- High-value settlement
- Zero-downtime requirements
- Distributed trust

Requires:
- Multiple evaluator deployments
- Quorum coordination logic
- Agreement verification

### 8.2 Operational Recommendations

**Mirror diversity:**
- Use Epoch Clock mirrors across different jurisdictions
- Monitor mirror agreement rates
- Alert on divergence

**Evaluator diversity:**
- Geographic distribution (reduces single-jurisdiction risk)
- Administrative separation (reduces correlation risk)
- Implementation diversity (reduces bug correlation)

**Monitoring:**
- Track tick freshness
- Track mirror agreement success rate
- Track evaluator uptime
- Track authorization success/failure rates
- Alert on anomalies

**Incident response:**
- Runbooks for clock divergence
- Runbooks for evaluator failure
- Escalation procedures for LOCKED outcomes
- Communication plan for downtime

**Operator education:**
- Training on fail-closed behavior
- Understanding of sovereignty preservation
- Awareness that refusal is not an error

### 8.3 Testing and Validation

Implementations MUST provide evidence demonstrating:

* Deterministic evaluation (same inputs produce same outputs)
* Cross-platform equivalence
* Canonical encoding stability (byte-stable encode/decode/encode)
* Proper error handling for all error codes
* Replay prevention
* Expiry enforcement
* Binding verification

---

## 9. Security Considerations

### 9.1 Threats BPC Prevents

- Premature transaction construction before authorization
- Replay of authorization outside validity windows
- Post-authorization mutation of transaction structure
- Execution under stale or ambiguous evidence
- Partial execution during evaluation failures
- Ambiguity-driven downgrade attacks

### 9.2 Threats BPC Does NOT Prevent

- Key compromise outside BPC scope
- Coerced or malicious signers
- Miner censorship or reordering (use Seal-360 for mempool protection)
- Network denial-of-service
- Side-channel attacks on implementation
- Legal or contractual disputes
- Subjective condition evaluation

### 9.3 Attacker Capabilities (Assumed)

- Observe network traffic and mempool contents
- Replay or substitute artifacts between protocol phases
- Present stale, partial, or manipulated evidence
- Exploit time ambiguity or clock skew
- Compromise individual mirrors or evaluators
- Exploit implementation vulnerabilities

### 9.4 Attacker Limitations (Assumed)

- Cannot forge cryptographic signatures
- Cannot alter deterministic evaluation logic without detection
- Cannot bypass execution gate without non-conformance
- Cannot compromise Epoch Clock profile inscription (immutable on Bitcoin blockchain)

### 9.5 Template Hash Function (Normative)

The template hash binds an authorization outcome to a specific, immutable transaction instance.

This hash is an **execution binding**: it commits to the exact transaction bytes that will be broadcast if execution proceeds. It is distinct from the `PSBTTemplateCanonical` hash used to bind authorization to a transaction template.

```text
template_hash = SHA256(raw_tx_bytes)
```

**Where:**

* `raw_tx_bytes` is the exact Bitcoin wire serialization of the transaction that would be constructed and broadcast after authorization.
* `SHA256` denotes a single application of the SHA-256 hash function.

**Requirements:**

* The hash function MUST be SHA-256.
* Double-SHA256 (SHA256d) MUST NOT be used.
* Alternative hash functions (including SHAKE256) MUST NOT be used for this execution binding.

**Rationale:**

* SHA-256 is Bitcoin’s canonical hash function and is universally available across wallet, signer, and auditor implementations.
* The security goal here is integrity binding between authorization and execution, not proof-of-work compatibility or domain separation.
* A single-round SHA-256 avoids unnecessary implementation variance while providing a stable 32-byte commitment.

Any deviation from this hash construction MUST be treated as an execution binding failure and MUST result in refusal at the execution gate.

### 9.6 Canonical Transaction Serialization (Normative)

The template hash MUST be computed over a canonical transaction byte sequence.

**Serialization rules:**

- The transaction MUST use standard Bitcoin wire serialization.
- The following fields MUST be included exactly as serialized for broadcast:
  - `version` (4 bytes, little-endian)
  - all inputs:
    - outpoint (`txid`, `vout`)
    - `scriptSig`
    - `sequence`
  - all outputs:
    - `value` (8 bytes, little-endian)
    - `scriptPubKey`
  - witness data (if present)
  - `locktime` (4 bytes, little-endian)

**Forbidden transformations:**
- Witness stripping or reordering
- Signature normalization or rewriting
- Input or output reordering
- Fee mutation
- Any byte-level rewrite after hashing

The byte sequence used for hashing MUST be identical to the byte sequence
that would be broadcast if execution proceeds.

Any divergence between the hashed bytes and the constructed transaction
bytes MUST result in refusal at the execution gate.

**Rationale:**
Authorization binding is only meaningful if the authorized structure and
the executed structure are byte-identical.

### 9.7 Fee Binding and Failure Domain Separation (Normative)

This section defines deterministic fee handling and clarifies the distinction
between **evaluation failures** and **execution-gate failures**.

---

#### 9.7.1 Fee Rate Binding

If `fee_rate_sat_vb` is specified in `PSBTTemplateCanonical`, it is **binding**.

**Binding rule:**

```

abs(actual_fee_rate - fee_rate_sat_vb) ≤ 1 sat/vB

```

- The tolerance is fixed at **±1 sat/vB**.
- Percentage-based tolerances are NOT permitted.
- Implementations MUST use the same fee-rate calculation method when
  evaluating and executing.

If `fee_rate_sat_vb` is `null`, fee rate is not binding and MAY be chosen
at construction time.

Any transaction exceeding the allowed tolerance MUST be refused at the
execution gate.

---

#### 9.7.2 Failure Domain Separation

BPC defines two distinct failure domains with different semantics.

##### Evaluation Failures (Evaluator Domain)

Evaluation failures occur during deterministic authorization.

- Result in `DENY` or `LOCKED`
- No transaction construction permitted
- No execution attempt allowed

Examples:
- Missing or invalid evidence
- Failed approval verification
- Expired intent
- Time ambiguity or clock divergence

These failures are final for the given intent and evidence set.

---

##### Execution-Gate Failures (Executor Domain)

Execution-gate failures occur **after** a valid `ALLOW` outcome has been issued.

- Result in **REFUSE**
- Transaction construction or signing is blocked
- Authorization itself remains valid but unusable

Examples:
- Outcome expired
- Outcome replay detected
- Binding mismatch (intent, session, or template hash)
- Fee rate outside allowed tolerance
- Final transaction divergence

Execution-gate failures do NOT retroactively invalidate the evaluator decision.
They prevent unsafe execution under changed conditions.

---

#### 9.7.3 Error Classification

| Error Type | Domain | Action |
|-----------|--------|--------|
| `DENY` | Evaluator | Abort authorization |
| `LOCKED` | Evaluator | Fail closed |
| `E_OUTCOME_EXPIRED` | Execution gate | Refuse execution |
| `E_OUTCOME_REPLAYED` | Execution gate | Refuse execution |
| `E_PSBT_TEMPLATE_MISMATCH` | Execution gate | Refuse execution |
| Fee tolerance violation | Execution gate | Refuse execution |

Implementations MUST NOT conflate evaluator failures with execution-gate
failures.

---

**Security Rationale:**

- Evaluators decide **whether** execution is permitted.
- Executors decide **whether it is still safe** to execute.

Separating these domains preserves determinism, auditability, and fail-closed
semantics under changing execution conditions.

---

### Annex A: Deterministic Encoding (Normative)

#### A.1 BPC-Native Objects

All BPC-native objects that are hashed or signed MUST use Deterministic CBOR (DetCBOR).

**Canonical requirements:**

* **Key ordering**: Keys MUST be sorted lexicographically by their encoded bytes.
* **Definite-length encoding**: Indefinite-length arrays or maps MUST NOT be used.
* **Unique map keys**: Duplicate keys are prohibited.
* **No floating-point**: Only fixed-point integers are permitted.
* **Integer representation**: MUST use the smallest possible integer representation.

**Stability requirement:**
Decode-encode cycles MUST be byte-stable, such that `original_bytes == encode(decode(original_bytes))`.

#### A.2 Epoch Clock Objects

Epoch Clock artifacts MUST be treated as externally canonical:

* Artifacts MUST be stored and compared as opaque JCS JSON bytes.
* Epoch Clock objects MUST NOT be re-encoded into CBOR.
* Signature verification MUST be performed on the exact bytes received from the mirror.
* Mirror agreement requires byte-identical comparison.

#### A.3 Hash Function

Unless otherwise specified:

* **Hash function**: SHA-256.
* **Input**: Canonical DetCBOR bytes or JCS JSON for Epoch Clock artifacts.
* **Output**: 32-byte digest.

---

### Annex B: Multi-Evaluator Quorum (Normative)

**B.1 Input Consistency**

All evaluators in a quorum MUST evaluate byte-identical inputs. Input divergence is non-conformant and MUST be detected before evaluation.

**B.2 Decision Precedence**

Quorum decisions follow strict precedence:

1. Any **DENY** outcome → overall result is **DENY**.
2. Else, any **LOCKED** outcome → overall result is **LOCKED**.
3. Else, a quorum of **ALLOW** outcomes is required → overall result is **ALLOW**.

No voting, averaging, or reconciliation is permitted.

**B.3 Quorum Satisfaction**

A quorum is satisfied if M evaluators emit **ALLOW** with identical binding fields:

* `bound_intent_hash`
* `bound_contract_id`
* `bound_session_id`
* `bound_psbt_hash`

All outcomes MUST be unexpired at execution time.

**B.4 Binding Divergence**

Divergent binding fields among **ALLOW** outcomes MUST result in a fail-closed state, and execution MUST be refused. No selection or reconciliation is permitted among outcomes with different bindings.

**B.5 Quorum Configuration**

Quorum parameters are defined as:

* **M**: Minimum **ALLOW** outcomes required.
* **N**: Total evaluators queried.
* **M ≤ N**.

Typical configurations include M=2, N=3 (majority) or M=3, N=5 (strong majority).

---

### Annex C: ExecutionResult (Normative)

`ExecutionResult` records each execution attempt for audit and debugging.

**C.1 Structure**

```markdown
ExecutionResult = {
  version: 1,
  execution_id: bstr,
  decision_id: bstr,
  contract_id: bstr,
  intent_hash: bstr(32),
  status: "NOT_SENT" | "REFUSED" | "SENT" | "CONFIRMED" | "FAILED",
  txid: bstr(32) / null,
  error: ErrorObject / null,
  result_t: uint
}

```

**C.2 Status Semantics**

* **NOT_SENT**: Execution aborted before broadcast attempt due to user cancellation or pre-broadcast validation failure.
* **REFUSED**: Authorization or verification failure prevented construction, such as an expired outcome, replayed outcome, binding mismatch, or a DENY/LOCKED decision.
* **SENT**: Transaction successfully constructed, signed, and broadcast; `txid` MUST be present.
* **CONFIRMED**: Transaction observed in a block via optional blockchain monitoring.
* **FAILED**: Construction or signing succeeded but broadcast failed; the transaction exists but was not sent to the network.

**C.3 Invariants**

* An `ExecutionResult` MUST be produced for every execution attempt.
* `ExecutionResult` is immutable once created.
* The `result_t` MUST be derived from Epoch Clock only, with no reliance on system time.
* No status transition is permitted from REFUSED or FAILED to SENT.

**C.4 Error Object**

```markdown
ErrorObject = {
  code: tstr,              // Error code (e.g., "E_OUTCOME_EXPIRED")
  message: tstr,           // Human-readable description
  context: map / null      // Additional debugging context
}

```
---

## Annex D: Privacy Considerations (Informative)

BPC provides no privacy by default. All protocol artifacts are observable.

### D.1 Observable Information

- Intent structure (inputs, outputs, amounts)
- Contract identifiers and session IDs
- Evaluation outcomes and timestamps
- Approval signatures and approver identities
- Proof content (unless encrypted or zero-knowledge)

### D.2 Privacy Enhancement Techniques

Privacy MAY be added via:

**Encrypted intent bodies:**
- Encrypt intent_body field
- Evaluator decrypts with authorized key
- Preserves verifiability while hiding content

**Zero-knowledge proofs:**
- Replace approval signatures with ZK proofs of authorization
- Replace payment proofs with ZK proofs of payment
- Preserves verification without revealing details

**Private transport:**
- Use Tor or VPN for communication with evaluators
- Use encrypted channels (TLS)
- Avoid correlation across intents

**Evaluator privacy properties:**
- Some evaluator implementations may provide privacy guarantees
- Evaluator-specific privacy documentation should be consulted

---

## Annex E: Fee Market Considerations (Informative)

Fee volatility may impact authorization-to-execution timing.

### E.1 Fee Handling Strategies

**Strategy 1: Short validity windows**
- Keep valid_until_t close to current_t (e.g., 1 hour)
- Reduces exposure to fee market changes
- Requires quick execution after authorization

**Strategy 2: Fee buffers**
- Build fee slack into PSBT template
- Accept higher fees than minimum required
- Accommodates fee market increases

**Strategy 3: Template-bound RBF**
- Enable RBF via sequence values in template
- Allows fee bumping after authorization
- Preserves authorization validity

**Strategy 4: Conservative estimation**
- Use high fee estimates when creating intent
- Reduces risk of insufficient fees
- May overpay in stable fee markets

### E.2 Fee Rate Binding

If PSBTTemplateCanonical includes binding fee_rate_sat_vb:

- Authorization is specific to that fee rate (within tolerance)
- Higher fees require new authorization
- Prevents unexpected cost increases

If fee_rate_sat_vb is null:

- Any fee rate is authorized
- Executor determines fee at construction time
- Risk of unexpectedly high fees if market volatile

Recommendation: Use binding fees for high-value settlement.

---

## Annex F: Non-Conformant Patterns (Informative)

The following patterns are explicitly non-conformant and MUST be avoided:

**Treating BPC as on-chain execution:**
- BPC is pre-construction authorization, not Script or consensus
- Do not expect on-chain enforcement of BPC rules

**Treating ALLOW as inclusion guarantee:**
- ALLOW authorizes construction, not blockchain confirmation
- Miners may still refuse or delay inclusion

**Treating evaluators as truth oracles:**
- Evaluators verify evidence, not assert external facts
- Evidence must be independently verifiable

**Claiming BPC is quantum-safe Bitcoin replacement:**
- BPC is authorization, not cryptographic hardening
- Use BIP-360 for output-layer quantum resistance
- Use Seal-360 for execution-layer quantum resistance

**Using BPC for dispute resolution:**
- BPC prevents unauthorized construction
- BPC does not resolve subjective disputes or legal claims

**Removing signer sovereignty:**
- Automating signing without user verification
- Bypassing hardware wallet verification
- Proceeding on behalf of key holder

**Allowing construction before ALLOW:**
- Constructing transaction "optimistically"
- Caching signed transactions for future authorization
- Any speculation on authorization result

**Using system time instead of Epoch Clock:**
- Wall-clock fallback when Epoch Clock unavailable
- System time for expiry checking
- Any time source other than verified Epoch Clock ticks

---

### Annex G: Implementation Classes (Informative)

**G.1 Wallet-Only Enforcement**

* **Architecture**: Evaluation logic runs locally within the wallet without an external evaluator; the user configures all conditions.
* **Suitable for**: Personal custody scenarios, simple time-based releases, and single-party authorization.
* **Not suitable for**: Multi-party settlement, external proof verification (e.g., payment confirmations, oracles), or high-assurance auditable settlement.
* **Security properties**: No evaluator trust is required as the user controls all evaluation logic, though verification is limited to conditions the wallet can locally process.

**G.2 External Evaluator + Wallet**

* **Architecture**: An evaluator service performs the evaluation; the wallet submits the intent and evidence, then independently verifies the signed outcome before the signer performs a final check.
* **Suitable for**: OTC settlement with external parties, treasury operations with policy enforcement, and multi-party approval workflows.
* **Requirements**: Trusted evaluator deployment, secure evaluator key management, and wallet integration via API or protocol.
* **Security properties**: The evaluator provides a verification service, but the wallet and signer remain sovereign; the evaluator cannot construct or sign transactions.

**G.3 Multi-Evaluator Quorum**

* **Architecture**: M-of-N evaluators must emit ALLOW; the wallet collects all outcomes and quorum logic verifies agreement and binding consistency.
* **Suitable for**: High-value settlement requiring reduced single-evaluator trust, zero-downtime requirements, and geographic or jurisdictional diversity.
* **Requirements**: Multiple evaluator deployments, quorum coordination logic in the wallet, and agreement verification implementation.
* **Security properties**: Tolerates individual evaluator failure or compromise; requires M evaluators to collude for incorrect authorization; provides higher availability.
* **Trade-offs**: Increased latency, coordination complexity, and operational overhead.

---

### Annex H: Future Extensions (Informative)

**H.1 Zero-Knowledge Evidence**
Replace approvals and proofs with zero-knowledge proofs:

* Prove authorization without revealing approver identity.
* Prove payment without revealing amount or parties.
* Preserves verification while enhancing privacy.

**H.2 Threshold Signatures**
Integration with threshold signature schemes:

* Approval expressed as threshold signature shares.
* Combines authorization and signing in single operation.
* Reduces round-trips in multi-party scenarios.

**H.3 Cross-Chain Bridge Discipline**
Use BPC to govern cross-chain transfers:

* Intent includes source and destination chains.
* Evidence includes cross-chain proofs.
* Prevents premature withdrawal before deposit confirmed.

**H.4 Hardware Wallet Protocol Extensions**
Dedicated hardware wallet protocols for BPC:

* Native BPC object display and verification.
* Secure outcome storage and replay prevention.
* User-friendly authorization review.

**H.5 Richer Attestation Types**
Support for additional evidence types:

* Government-issued digital identity attestations.
* Corporate authorization signatures.
* Notarized documents.
* Regulatory compliance proofs.

---

### Annex I: Determinism Verification (Informative)

#### I.1 Example Intent Parameters

A `PreContractIntent` MAY be constructed with:

* A fixed `contract_id`
* A fixed `session_id`
* An `expiry_t` expressed as an Epoch Clock tick
* A `PSBTTemplateCanonical` containing:

  * One input
  * One output
  * No signatures
  * No witnesses
  * `fee_rate_sat_vb = null`

The intent MUST be encoded using Deterministic CBOR with stable key ordering and definite-length encoding.

---

#### I.2 Deterministic Properties

* Encoding the same logical intent data MUST produce byte-identical canonical bytes.
* Hashing those canonical bytes MUST produce a stable `intent_hash`.
* Re-encoding the decoded object MUST produce identical bytes.
* Evaluation of the same intent and evidence inputs MUST produce the same decision outcome.

Any deviation is non-conformant.

---

#### I.3 Evaluation Behavior

Given:

* Valid Epoch Clock time evidence not exceeding `expiry_t`
* No required approvals
* No required external proofs

A conformant evaluator MUST emit:

```
decision = ALLOW
```

If time evidence is unavailable, unverifiable, or exceeds `expiry_t`, evaluation MUST emit `LOCKED` or `DENY` as defined in the core protocol.

---

#### I.4 Verification Checklist

* Deterministic CBOR encoding stability
* Canonical ordering of PSBT templates
* Separation between intent hashing and execution binding
* Absence of system-time dependence in deterministic evaluation
* Fail-closed behavior under missing or ambiguous inputs

---

### Normative References

* **RFC 2119:** Key words for RFCs to Indicate Requirement Levels
* **RFC 8949:** Concise Binary Object Representation (CBOR)
* **RFC 8785:** JSON Canonicalization Scheme (JCS)
* **BIP 174:** Partially Signed Bitcoin Transactions
* **BIP 69:** Lexicographical Indexing of Transaction Inputs and Outputs
* **Epoch Clock v2.0.0:** Pinned profile (`ordinal:439d7ab1972803dd984bf7d5f05af6d9f369cf52197440e6dda1d9a2ef59b6ebi0`)

---

### Informative References

* **Seal-360:** Execution-Layer Mitigation for BIP-360
* **BIP-360:** Pay-to-Tapscript-Hash specification

---

### Acknowledgments

Bitcoin and PSBT contributor communities, CBOR/JCS canonical encoding standards, Epoch Clock design contributors, BIP-360 team for quantum-resistant outputs, and the post-quantum cryptography research community.

---

Support this work:
bc1q380874ggwuavgldrsyqzzn9zmvvldkrs8aygkw
