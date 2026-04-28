# Formal Verification Strategy

**Document ID:** SEC-FV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the formal verification strategy for SBOP's critical security and safety properties. Formal verification provides mathematical proof that the boot state machine, verification logic, and security invariants cannot be violated — going beyond testing to exhaustively cover all possible states and inputs.

---

## 2. Verification Scope

### 2.1 In-Scope

| Target | Rationale |
| --- | --- |
| Boot state machine | Single point of trust; any flaw allows untrusted execution |
| Signature verification logic | Correctness of verification decision |
| Anti-rollback enforcement | Prevents replay of vulnerable firmware |
| Slot selection logic | Must never select unverified slot |
| Debug lock state machine | Controls access to entire device |
| Key zeroization on tamper | Critical safety barrier |

### 2.2 Out-of-Scope

| Target | Rationale | Verification Approach |
| --- | --- | --- |
| Cryptographic algorithm correctness | Assumed from standard implementations | Crypto compliance testing (TEST-BOOT-002) |
| OTA download protocol | Stateful network protocol; unbounded state space | Model-based testing |
| Application firmware | Not SBOP domain | Application-level V&V |
| Hardware behavior (TRNG, OTP, flash) | Platform-specific | Hardware validation testing |

---

## 3. Formal Properties

### 3.1 Liveness Properties

**P-LIVE-01: Successful Boot Progress**

```
AG (RESET → AF EXECUTE)
```

From RESET state, on all paths where verification succeeds, EXECUTE is eventually reached.

**P-LIVE-02: FAILSAFE Recovery**

```
AG (FAILSAFE → AF (EXECUTE ∨ TAMPER_LOCK))
```

From FAILSAFE, the system eventually either recovers (back to EXECUTE) or permanently locks.

### 3.2 Safety Properties (Invariants)

**P-SAFE-01: No Execution Without Full Verification**

```
AG ¬((state = EXECUTE) ∧ ¬(verify_sig = PASS ∧ verify_hash = PASS ∧ check_ver = PASS))
```

It is impossible to reach EXECUTE without passing signature verification, hash verification, and version check.

**P-SAFE-02: All Verification Failures Lead to FAILSAFE**

```
AG ((verify_sig = FAIL ∨ verify_hash = FAIL ∨ check_ver = FAIL) → AF state = FAILSAFE)
```

**P-SAFE-03: Slot Integrity Invariant**

```
AG ((slot[A].status = ACTIVE → slot[A].was_verified = true) ∧
    (slot[B].status = ACTIVE → slot[B].was_verified = true))
```

No slot can be marked ACTIVE unless it has passed verification.

**P-SAFE-04: Version Monotonicity**

```
AG (new_version > current_version) ∨ ¬(version_commit_allowed)
```

The version counter only increases; no transition decrements it.

**P-SAFE-05: No State Skip**

```
AG ¬(state_jump_allowed)
```
Where `state_jump_allowed` is true if any transition bypasses a required intermediate state:

```
RESET → INIT → (SELECT_SLOT | FAILSAFE)
SELECT_SLOT → (LOAD_IMAGE | FAILSAFE)
LOAD_IMAGE → (VERIFY_SIGNATURE | FAILSAFE)
VERIFY_SIGNATURE → (VERIFY_INTEGRITY | FAILSAFE)
VERIFY_INTEGRITY → (CHECK_VERSION | FAILSAFE)
CHECK_VERSION → (COMMIT_VERSION | FAILSAFE)
COMMIT_VERSION → (MARK_ACTIVE | FAILSAFE)
MARK_ACTIVE → (LOCK_BOOT | FAILSAFE)
LOCK_BOOT → EXECUTE
```

**P-SAFE-06: Key Material Isolation**

```
AG ¬(debug_unlocked ∧ key_accessible)
```

Key material is never accessible via debug, regardless of debug unlock state.

**P-SAFE-07: Tamper Response Invariant**

```
AG (tamper_detected → AF keys_zeroized)
```

**P-SAFE-08: One Active Slot Maximum**

```
AG ¬(slot[A].status = ACTIVE ∧ slot[B].status = ACTIVE)
```

At most one slot can be ACTIVE at any time.

### 3.3 Temporal Properties (LTL)

**P-TEMP-01: Verification Before Execution**

```
G(¬EXECUTE U VERIFICATION_COMPLETE)
```

EXECUTE cannot hold until VERIFICATION_COMPLETE holds.

**P-TEMP-02: Tamper Persistence**

```
G(tamper_detected → G(locked))
```

Once tamper is detected, the device remains locked permanently.

---

## 4. Formal Methods by Target

### 4.1 Boot State Machine — Model Checking

**Tool:** TLA+ (Temporal Logic of Actions) or nuXmv

**Approach:**
1. Model boot state machine as finite state automaton
2. Express properties P-SAFE-01 through P-SAFE-08 as invariants and LTL formulas
3. Run model checker to exhaustively explore state space
4. For any counterexample: fix model or specification

**State Space Bounds:**

| Variable | Domain | Size |
| --- | --- | --- |
| state | {RESET, INIT, SELECT_SLOT, LOAD_IMAGE, VERIFY_SIGNATURE, VERIFY_INTEGRITY, CHECK_VERSION, COMMIT_VERSION, MARK_ACTIVE, LOCK_BOOT, EXECUTE, FAILSAFE, TAMPER_LOCK} | 13 |
| verify_sig | {NONE, IN_PROGRESS, PASS, FAIL} | 4 |
| verify_hash | {NONE, IN_PROGRESS, PASS, FAIL} | 4 |
| check_ver | {NONE, IN_PROGRESS, PASS, FAIL} | 4 |
| slot[A].status | {EMPTY, INACTIVE, PENDING, ACTIVE, INVALID} | 5 |
| slot[B].status | {EMPTY, INACTIVE, PENDING, ACTIVE, INVALID} | 5 |
| version_counter | 0..2^32-1 | symbolic |
| tamper_detected | {TRUE, FALSE} | 2 |
| debug_unlocked | {TRUE, FALSE} | 2 |

**Total explicit state space (bounded):** 13 × 4^3 × 5^2 × 2^2 = 13 × 64 × 25 × 4 = 83,200 states
(Version counter treated symbolically — not enumerated)

### 4.2 Signature Verification Logic — Theorem Proving

**Tool:** Coq, Isabelle/HOL, or F*

**Approach:**
1. Formally specify ECDSA P-256 verification algorithm in proof assistant
2. Prove that `verify_signature(msg, sig, pk) = true ⇔ sig = Sign(sk, msg)`
3. Prove that `verify_signature` is constant-time (no data-dependent branches)
4. Extract verified C code via Coq's extraction or F*'s Low* to C

**Target Properties:**

| Property | Formal Statement | Tool |
| --- | --- | --- |
| Signature correctness | `∀ msg, sk. verify(msg, sign(msg, sk), pk(sk)) = true` | Coq |
| Signature security | `∀ msg, sig. verify(msg, sig, pk) = true → ∃ sk. sig = sign(msg, sk)` | Coq |
| Constant-time | `∀ m1, m2. |m1|=|m2| → time(verify(m1)) = time(verify(m2))` | F* |

### 4.3 Constant-Time Verification — Abstract Interpretation

**Tool:** ct-verif, FlowTracker, or SideTrail

**Approach:**
1. Annotate security-critical functions with secret labels (key material, hash output)
2. Run abstract interpreter to verify no secret-dependent:
   - Branch conditions
   - Memory access patterns
   - Instruction execution counts
3. Flag any violation as blocking for certification

**Functions Requiring Constant-Time Verification:**

| Function | Tool | Property |
| --- | --- | --- |
| `constant_time_compare(a, b, len)` | ct-verif | No early return; no secret-dependent branch |
| `verify_ecdsa(msg, sig, pk)` | SideTrail | No key-dependent execution time |
| `sha256_verify(msg, expected_hash)` | ct-verif | Compare step is constant-time |
| `check_version(new, current)` | Manual review + timing measurement | Fixed-sequence comparison |

### 4.4 Anti-Rollback — Model Checking

**Tool:** nuXmv or CBMC (C Bounded Model Checker)

**Properties:**
```
AG (version_counter' ≥ version_counter)        // never decreases
AG (install_allowed → new_version > version_counter)  // must be higher
```

### 4.5 Debug Lock State Machine — Model Checking

**Tool:** TLA+ or nuXmv

**States:**
```
UNLOCKED → PROVISIONING → LOCKED → (AUTH_CHALLENGE → UNLOCKED_TEMPORARY → LOCKED)
                                   → (PERMANENTLY_LOCKED)
```

**Properties:**
```
AG (PROVISIONING → AF LOCKED)                     // provisioning always locks
AG (fail_count ≥ 10 → AF PERMANENTLY_LOCKED)     // rate limit
AG ¬(PERMANENTLY_LOCKED ∧ debug_port_accessible)  // permanent lock = no access
AG ¬(UNLOCKED_TEMPORARY ∧ key_material_accessible) // keys always protected
```

---

## 5. Tool Selection Rationale

| Tool | Type | Target | Rationale |
| --- | --- | --- | --- |
| TLA+ | Model checker | State machine properties | Industry standard; TLC model checker; PlusCal for readability |
| nuXmv | Model checker | State machines + LTL | Strong for hardware-software co-verification |
| Coq | Proof assistant | Algorithm correctness | Extracted code; mature ecosystem; CompCert precedent |
| F* | Program verifier | Constant-time + correctness | Low* subset extracts to C; SMT-based automation |
| CBMC | Bounded model checker | C implementation verification | Direct C analysis; useful post-implementation |
| ct-verif | Side-channel verifier | Constant-time enforcement | Specialized for timing leaks |

---

## 6. Verification Workflow

### 6.1 Specification Phase (Current)

```
Boot_Flow_Pseudocode.md → TLA+ model → model check properties → fix counterexamples
```

**Deliverable:** TLA+ specification with all 8 safety properties proven.

### 6.2 Implementation Phase (Phase 1+)

```
C/Rust implementation → CBMC bounded check (loop unrolling) → ct-verif constant-time check
```

**Deliverable:** Bounded verification report (loop bounds, array bounds, no undefined behavior).

### 6.3 Certification Phase (Phase 7)

```
Final implementation → Full formal verification → Evidence package for auditor
```

**Deliverable:** Formal verification evidence for DO-178C DAL A / ISO 26262 ASIL D.

---

## 7. DO-178C Formal Methods Supplement (DO-333)

For DAL A certification with formal methods:

| DO-333 Activity | SBOP Approach | Evidence |
| --- | --- | --- |
| Formal specification | TLA+ model of state machine | TLA+ specification file |
| Formal analysis | Model checking with TLC | Counterexample-free report |
| Formal verification | CBMC of C implementation | Coverage report |
| Traceability | Properties → requirements → tests | Traceability matrix |
| Tool qualification | TLC tool qualification | Tool qualification records |

### 7.1 Formal Methods Credit

Using formal methods can reduce DO-178C verification objectives:

| Traditional Objective | Formal Methods Alternative | Credit |
| --- | --- | --- |
| MC/DC testing | Formal verification of same property | Can replace structural coverage for verified properties |
| Requirements-based testing | Exhaustive model checking | Can reduce test cases for properties covered |
| Stack analysis | Bounded model checking | Static guarantee of no stack overflow |

---

## 8. Property Coverage Matrix

| Property | P-SAFE-01 | P-SAFE-02 | P-SAFE-03 | P-SAFE-04 | P-SAFE-05 | P-SAFE-06 | P-SAFE-07 | P-SAFE-08 | P-LIVE-01 | P-LIVE-02 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| REQ-FR-BOOT-001 | ✓ | ✓ | — | — | ✓ | — | — | — | ✓ | — |
| REQ-FR-BOOT-002 | — | — | ✓ | — | — | — | — | — | — | — |
| REQ-FR-BOOT-004 | — | — | — | ✓ | — | — | — | — | — | — |
| REQ-SR-BOOT-001 | ✓ | ✓ | — | — | ✓ | — | — | — | — | — |
| REQ-SR-KEY-001 | — | — | — | — | — | ✓ | ✓ | — | — | — |
| REQ-SR-PHY-001 | — | — | — | — | — | — | ✓ | — | — | — |
| REQ-SR-DBG-001 | — | — | — | — | — | ✓ | — | — | — | — |
| REQ-FR-OTA-004 | — | — | — | — | — | — | — | ✓ | — | ✓ |

**Coverage:** 8 properties cover 8 of 10 security requirements. P-LIVE-01 and P-LIVE-02 additionally cover availability.
Gaps: REQ-SR-SC-001 (supply chain — not amenable to formal methods), REQ-SR-SCH-001 (side-channel — addressed by ct-verif).

---

## 9. Counterexample Analysis Template

For each counterexample found by the model checker:

| Field | Description |
| --- | --- |
| Trace ID | Unique identifier |
| Property violated | Which P-SAFE or P-LIVE property |
| State trace | Sequence of states leading to violation |
| Root cause | Why the model allowed the violation |
| Fix | Change to model or specification |
| Regression | Which properties re-verified after fix |

---

## 10. Assumptions and Limitations

### 10.1 Formalized Assumptions

| ID | Assumption | Justification |
| --- | --- | --- |
| ASM-01 | TRNG output is unpredictable with sufficient entropy | Per NIST SP 800-90B health tests |
| ASM-02 | OTP cells are truly write-once | Hardware specification; verified by fuse readback |
| ASM-03 | HSM prevents key extraction | FIPS 140-2 Level 3 certification |
| ASM-04 | ECDSA P-256 / Ed25519 provide existential unforgeability | Cryptographic assumption; reviewed algorithms |
| ASM-05 | SHA-256 is collision-resistant | Cryptographic assumption |
| ASM-06 | MPU/MMU correctly enforces access permissions | Hardware specification; verified by access tests |

### 10.2 Limitations

1. **Cryptographic assumptions**: Formal verification relies on cryptographic assumptions (ECDLP hardness, SHA-256 collision resistance). Breaking these at the mathematical level is out of scope.
2. **Hardware correctness**: Formal verification assumes hardware behaves according to specification. Hardware errata and physical attacks require separate analysis.
3. **Unbounded state**: The version counter is 32-bit — practically unbounded. Model checking uses symbolic representation.
4. **Side channels**: Timing verification addresses software-level leaks. Hardware-level EM/power side channels require physical measurement (TVLA).

---

## 11. References

| Document | Reference |
| --- | --- |
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| OTA Flow Pseudocode | `../03_Subsystem/Update/OTA_Flow_Pseudocode.md` |
| Interface Contracts | `../03_Subsystem/Interface_Contracts.md` |
| Side-Channel Countermeasures | `Side_Channel_Countermeasures.md` |
| Safety Analysis | `Safety_Analysis.md` |
| Attack Tree | `Attack_Tree.md` |
| Requirements | `../01_Requirements/` |
