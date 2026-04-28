# Functional Safety Analysis

**Document ID:** SEC-SAF-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document analyzes the relationship between SBOP security controls and functional safety, mapping SBOP's secure boot architecture to safety standards across automotive (ISO 26262), industrial (IEC 61508), aviation (DO-178C/DO-254), and medical (IEC 62304) domains.

Security is a prerequisite for safety in connected embedded systems: a compromised boot chain invalidates all safety assumptions about the executing software.

---

## 2. Safety-Standards Overview

| Standard | Domain | Safety Levels | Security Expectation |
| --- | --- | --- | --- |
| ISO 26262 | Automotive | ASIL A-D (QM) | ISO 21434 alignment required for ASIL B+ |
| IEC 61508 | Industrial | SIL 1-4 | Security considered in hazard analysis |
| DO-178C/DO-326A | Aviation | DAL A-E (SEAL 1-4) | Airworthiness security process (DO-326A) |
| IEC 62304 | Medical | Class A-C | Risk management per ISO 14971 |
| ECSS Q-ST-80 | Space | Cat A-D | Software product assurance |

### 2.1 Security-Safety Dependency

```
Security Failure → Compromised Firmware → Safety Function Failure → Hazard
```

If SBOP's security controls fail, the executing firmware is untrusted. Any safety function running on untrusted firmware has no safety integrity. Therefore, SBOP's security assurance level must be ≥ the highest safety integrity level of any function on the device.

---

## 3. Hazard Analysis

### 3.1 Top-Level Hazard

**HZ-001: Unauthorized firmware executes on device**

| Attribute | Value |
| --- | --- |
| Hazard | Execution of firmware that has not passed cryptographic verification |
| Potential harm | Uncontrolled system behavior leading to physical harm, mission loss, data breach |
| Safety barrier | SBOP secure boot with ECDSA/Ed25519 signature verification + SHA-256 integrity |
| Security barrier | HSM-backed signing, key hierarchy (KR→KD→KI), anti-rollback |

### 3.2 Hazard Causal Factors

| ID | Causal Factor | Safety Impact | SBOP Mitigation |
| --- | --- | --- | --- |
| HCF-01 | Signature verification skipped (fault injection) | ASIL D / SIL 4 / DAL A | Redundant verification + glitch detection |
| HCF-02 | Attacker signs malicious firmware | ASIL D / SIL 4 / DAL A | HSM + quorum signing |
| HCF-03 | Rollback to vulnerable firmware | ASIL C / SIL 3 / DAL B | Monotonic OTP version counter |
| HCF-04 | Firmware corrupted in storage | ASIL B / SIL 2 / DAL C | SHA-256 + slot CRC + write-verify |
| HCF-05 | Boot state machine corrupted (SEU) | ASIL D / SIL 4 / DAL A | Redundant state, fail-safe on inconsistency |
| HCF-06 | Debug port exposes secure assets | ASIL D / SIL 4 / DAL A | Debug lock + challenge-response auth |
| HCF-07 | Side-channel key extraction | ASIL C / SIL 3 / DAL B | Constant-time crypto, masking at SL 3+ |

---

## 4. Safety Integrity Level Mapping

### 4.1 ISO 26262 ASIL Decomposition

SBOP's boot verification path is the **single point of trust** for all post-boot software. ASIL decomposition is not applicable because there is no redundant, diverse verification path — SBOP is the sole arbiter of firmware authenticity.

| SBOP Function | Safety Relevance | Target ASIL | Rationale |
| --- | --- | --- | --- |
| Signature verification (boot) | If fails → uncontrolled SW | ASIL D (highest on device) | Single point of trust |
| Hash verification (boot) | If fails → corrupted SW | ASIL D | No independent check |
| Version check (anti-rollback) | If fails → vulnerable SW | ASIL C | Attacker must also compromise signing |
| OTA download + verify | If fails → corrupted update | ASIL C | Mitigated by boot-time re-verification |
| Slot state management | If fails → wrong slot booted | ASIL B | Detected by boot verification |
| Debug authentication | If fails → key exposure | ASIL D | Keys protect all safety functions |
| Tamper detection | If fails → undetected attack | ASIL C | Redundant detection at SL 3+ |

### 4.2 IEC 61508 SIL Mapping

| SBOP Function | Systematic Capability | Target SIL | Notes |
| --- | --- | --- | --- |
| Boot verification chain | SC 3 (SIL 3) | SIL 3 (SIL 4 with diversity) | Route 1S: proven in use + formal verification |
| Cryptographic operations | SC 3 | SIL 3 | Must use approved algorithms |
| OTA state machine | SC 2 | SIL 2 | Fallback to known-good state |
| Anti-rollback | SC 3 | SIL 3 | Monotonic counter with redundant check |

### 4.3 DO-178C DAL Mapping

| SBOP Function | DAL | Verification Objectives | Notes |
| --- | --- | --- | --- |
| Boot manager | DAL A | MC/DC on verification logic | 66 objectives for Level A |
| Crypto verification | DAL A | MC/DC | Must use DO-178C qualified crypto library or service history |
| OTA download | DAL C | Statement + decision coverage | Boot re-verification reduces criticality |
| Slot management | DAL C | Statement coverage | Failure detected at boot |

---

## 5. Safety Mechanisms

### 5.1 Detection Mechanisms

| ID | Mechanism | Detection Coverage | Fault Model |
| --- | --- | --- | --- |
| SM-DET-01 | Signature verification failure → FAILSAFE | 100% of unauthorized code | Systematic + random HW |
| SM-DET-02 | Hash mismatch → FAILSAFE | 100% of corruption (detectable) | Random HW (SEU, flash degradation) |
| SM-DET-03 | Version monotonicity violation → FAILSAFE | 100% of rollback | Systematic (attack) |
| SM-DET-04 | Slot metadata CRC mismatch → slot marked INVALID | 100% of metadata corruption | Random HW |
| SM-DET-05 | Glitch counter threshold → tamper response | Depends on sensor quality | Fault injection |
| SM-DET-06 | Watchdog timeout during boot → reset | Depends on WD configuration | Systematic + random HW |
| SM-DET-07 | Tamper switch → immediate response | Depends on switch placement | Physical intrusion |

### 5.2 Reaction Mechanisms

| ID | Reaction | Trigger | Safe State |
| --- | --- | --- | --- |
| SM-REACT-01 | Enter FAILSAFE | Any boot verification failure | No application code executes |
| SM-REACT-02 | Key zeroization | Critical tamper detected | Keys destroyed, device inoperative |
| SM-REACT-03 | Slot switch (A→B or B→A) | Repeated boot failure | Fallback to known-good image |
| SM-REACT-04 | Backend-forced update | FAILSAFE with connectivity | Recovery path via OTA |
| SM-REACT-05 | Permanent debug lock | Auth failure threshold | Debug port unrecoverable |

### 5.3 Safe State Definition

The SBOP safe state hierarchy:

| Level | State | Description | Exit Condition |
| --- | --- | --- | --- |
| 1 (Preferred) | EXECUTE | Verified firmware running | N/A — operational |
| 2 (Degraded) | FALLBACK | Running previous known-good firmware | Next successful OTA |
| 3 (Recoverable) | FAILSAFE | No firmware running, awaiting recovery | Backend-initiated OTA |
| 4 (Unrecoverable) | TAMPER_LOCK | Keys destroyed, device permanently locked | Physical replacement |

---

## 6. Systematic Fault Avoidance

### 6.1 Development Process Requirements

| Measure | ISO 26262-6 | IEC 61508-3 | DO-178C | SBOP Approach |
| --- | --- | --- | --- | --- |
| Semi-formal specification | Recommended for ASIL C+ | Recommended for SIL 3+ | Required for DAL A | Pseudocode with pre/post conditions |
| Formal verification | Recommended for ASIL D | Recommended for SIL 4 | Optional (DO-333) | Model checking of state machine |
| Defensive implementation | Required ASIL B+ | Required SIL 2+ | N/A | Input validation, no undefined behavior |
| Coding guidelines | MISRA C required | Recommended | Required | MISRA C:2012 Dir 4.14 compliance |
| Static analysis | Required ASIL B+ | Required SIL 2+ | Required | Zero warnings on critical path |

### 6.2 Security Measures as Safety Measures

| Security Measure | Safety Benefit | Standards Addressed |
| --- | --- | --- |
| Constant-time crypto | Prevents timing oracle → prevents key extraction → prevents unauthorized signing | ISO 26262, IEC 61508 |
| Redundant verification | Detects fault injection → prevents verification bypass | ISO 26262, DO-178C |
| Dual-image A/B | Enables recovery → maintains availability | IEC 61508, ECSS |
| MPU/MMU zone isolation | Prevents application from corrupting boot | IEC 62304, ISO 26262 |
| Tamper detection + response | Detects physical attacks → enters safe state | ISO 26262, IEC 62443 |
| SBOM + reproducible builds | Enables incident response → limits compromised firmware impact | ISO 21434, DO-326A |

---

## 7. Fault Tree Analysis (Safety Perspective)

```
Hazard: Unauthorized firmware executes
│
├── AND ── Signature verification fails
│   ├── OR ── Verification skipped (fault injection) [HCF-01]
│   ├── OR ── Attacker has valid signing key [HCF-02]
│   └── OR ── Crypto algorithm broken [RRES-SCH-001]
│
├── AND ── Integrity check fails
│   ├── OR ── Hash collision [mathematically infeasible for SHA-256]
│   ├── OR ── Hash verification bypassed [HCF-01]
│   └── OR ── Constant-time compare defeated [RRES-SCH-001]
│
├── AND ── Rollback protection fails
│   ├── OR ── Version counter corruption [HCF-03]
│   └── OR ── Version check bypassed [HCF-01]
│
└── AND ── Firmware image corrupted
    ├── OR ── Flash bit-flip (SEU) [SM-DET-02]
    ├── OR ── OTA corruption [SM-DET-04]
    └── OR ── Manufacturing defect [RRES-MFR-003]
```

**Minimum cut set analysis**: All top-level gates are AND — multiple independent failures required for hazard. The highest-risk cut set is {HCF-01, HCF-02} (fault injection that defeats redundant verification AND compromised signing key).

---

## 8. Proven-In-Use Argument (IEC 61508 Route 1S)

For SIL 3 via Route 1S (proven in use):

| Element | Evidence Required | Status |
| --- | --- | --- |
| ECDSA P-256 verification | Widespread deployment, no known breaks | Accepted primitive |
| SHA-256 | Widespread deployment, no practical collisions | Accepted primitive |
| Dual-image boot model | Deployed in safety-critical systems (e.g., UEFI Secure Boot in automotive ECUs) | Partial — requires platform-specific evidence |
| Constant-time implementation | Requires per-platform timing measurement | TBD during implementation |

---

## 9. Tool Qualification

| Tool | Standard | Qualification Required | Approach |
| --- | --- | --- | --- |
| C compiler | DO-178C / ISO 26262 | Yes for DAL A / ASIL D | Qualified compiler or output verification |
| `sbop-sign` (host) | DO-178C | Yes if used to sign flight software | Tool qualification per DO-330 |
| `sbop-verify` (host) | DO-178C | Yes if part of verification | Tool qualification per DO-330 |
| `sbop-pack` (host) | All | No (output independently verified) | Verification tool exemption |
| Model checker | DO-178C / IEC 61508 | Yes for DO-178C formal methods | DO-333 qualification |
| Static analyzer | All | Yes | Tool qualification or manual review of results |

---

## 10. Medical-Specific Considerations

### 10.1 IEC 62304 Risk Control

SBOP as SOUP (Software of Unknown Provenance) for medical device manufacturers:

| IEC 62304 Clause | Requirement | SBOP Evidence |
| --- | --- | --- |
| 5.3.4 (Risk control) | SOUP anomaly resolution process | Incident response + CVE monitoring |
| 5.3.5 (Segregation) | SOUP must be segregated from safety functions | Zone 1/2 isolation + MPU enforcement |
| 8.1.2 (Maintenance) | SOUP monitoring throughout lifecycle | Monitoring and telemetry |

### 10.2 ISO 14971 Residual Risk

Medical device manufacturers must assess whether the residual risk of SBOP security failure is acceptable for their device's intended use. This analysis must include:

1. Probability of SBOP compromise (see TARA)
2. Severity of harm if SBOP fails and malicious firmware executes
3. Risk/benefit analysis per ISO 14971 Clause 7

---

## 11. Domain-Specific Safety Constraints

### 11.1 Automotive (ISO 26262)

| Constraint | Impact on SBOP |
| --- | --- |
| ASIL D requires latent fault metric ≥ 99% | SBOP boot must complete within fault-tolerant time interval (FTTI) |
| Freedom from interference | Zone 1 must be protected from Zone 2 via MPU (no shared memory for critical data) |
| ASIL decomposition (ASIL D → ASIL C(D) + ASIL A(D)) | Not applicable — SBOP is single point of trust |

### 11.2 Space (ECSS)

| Constraint | Impact on SBOP |
| --- | --- |
| SEU immunity for boot state | Triple-modular redundancy (TMR) for boot state variables |
| TID degradation of flash | Slot wear-leveling, periodic slot integrity scrub |
| No ground intervention possible | Recovery boot must work autonomously |
| Long mission duration (15+ years) | Crypto agility required — algorithm migration path |

### 11.3 Aviation (DO-178C/DO-326A)

| Constraint | Impact on SBOP |
| --- | --- |
| MC/DC for Level A | Boot verification logic must achieve MC/DC coverage |
| DO-326A SEAL assignment | Security assurance level linked to DAL |
| Airworthiness security process | SBOP integrated into aircraft security risk assessment |
| CAST-32A multi-core | Boot on single locked core; other cores held in reset |

---

## 12. Safety Case Structure

```
G1: SBOP ensures only authentic firmware executes
├── G1.1: Signature verification is correct
│   ├── Sn1.1.1: ECDSA/Ed25519 algorithms are mathematically sound
│   ├── Sn1.1.2: Implementation is constant-time (TIM-001, TIM-002 evidence)
│   └── Sn1.1.3: Verification state machine has no bypass paths (formal verification)
├── G1.2: Integrity verification is correct
│   ├── Sn1.2.1: SHA-256 has no practical collisions
│   ├── Sn1.2.2: Hash comparison is constant-time (TIM-001)
│   └── Sn1.2.3: Image format parsing has no exploitable bugs (fuzz results)
├── G1.3: Anti-rollback is effective
│   ├── Sn1.3.1: OTP counter is write-once (hardware guarantee)
│   └── Sn1.3.2: Version check is redundant (FI-004)
├── G1.4: Key material is protected
│   ├── Sn1.4.1: Keys in secure element / TEE
│   ├── Sn1.4.2: No key access via debug
│   └── Sn1.4.3: Tamper detection triggers zeroization
└── G1.5: Failures lead to safe state
    ├── Sn1.5.1: All verification failures → FAILSAFE
    ├── Sn1.5.2: FAILSAFE prevents application execution
    └── Sn1.5.3: Recovery path exists (dual-image, backend OTA)
```

---

## 13. References

| Document | Reference |
| --- | --- |
| Attack Tree | `Attack_Tree.md` |
| TARA Methodology | `TARA_Methodology.md` |
| Residual Risk Register | `Residual_Risk.md` |
| Mitigation Strategy | `Mitigation_Strategy.md` |
| Side-Channel Countermeasures | `Side_Channel_Countermeasures.md` |
| Physical Tamper Resistance | `Physical_Tamper_Resistance.md` |
| ISO 21434 Detailed Mapping | `../07_Compliance/ISO_21434_Detailed_Mapping.md` |
| Aviation Compliance | `../07_Compliance/Aviation_Compliance_Mapping.md` |
| Medical Compliance | `../07_Compliance/Medical_Compliance_Mapping.md` |
| ECSS Compliance | `../07_Compliance/ECSS_Compliance_Mapping.md` |
