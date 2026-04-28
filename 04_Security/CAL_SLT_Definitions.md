# CAL and SL-T Definitions (IEC 62443 / ISO 21434)

**Document ID:** SEC-CAL-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the target Conformance Assessment Levels (CAL) per IEC 62443 and the Security Level Targets (SL-T) for each SBOP zone and subsystem. It establishes the criteria against which SBOP's security posture is measured.

---

## 2. CAL Definition (IEC 62443)

### 2.1 CAL Overview

Conformance Assessment Levels define the rigor of the assessment process applied to verify that security requirements are met.

| CAL | Name | Assessment Method | Independence | Penetration Testing |
| --- | --- | --- | --- | --- |
| CAL 1 | Self-Assessment | Internal review against requirements | None | Not required |
| CAL 2 | Independent Assessment | Assessment by independent organization | Required | Not required |
| CAL 3 | Independent Assessment with Penetration Testing | Assessment by independent organization | Required | Required (defined scope) |
| CAL 4 | Comprehensive Assessment | Assessment by independent organization | Required | Required (comprehensive) |

### 2.2 SBOP CAL Targets

| Subsystem | CAL Target | Rationale |
| --- | --- | --- |
| Boot Subsystem | **CAL 3** | Root of trust enforcement requires penetration testing validation |
| Crypto Subsystem | **CAL 3** | Cryptographic correctness requires independent verification |
| Identity Subsystem | **CAL 2** | Identity management; independent review without pen testing |
| Update Subsystem | **CAL 3** | OTA is primary attack vector; requires pen testing |
| Backend (overall) | **CAL 3** | Fleet management; independent assessment with pen testing |
| Toolchain | **CAL 2** | Build pipeline integrity; independent review |

### 2.3 CAL Achievement Roadmap

| Phase | CAL Target | Timeline | Prerequisites |
| --- | --- | --- | --- |
| Phase 1: Design Review | CAL 1 | Current | Documented requirements and design |
| Phase 2: Independent Review | CAL 2 | Post-Phase 14 | Subsystem implementation |
| Phase 3: Penetration Testing | CAL 3 | Post-Phase 2 | Operational test environment |
| Phase 4: Comprehensive | CAL 4 | Future (optional) | Full system deployment data |

---

## 3. Security Level (SL) Definitions

### 3.1 IEC 62443 Security Levels

| SL | Name | Attacker Profile | Resources | Skills | Motivation |
| --- | --- | --- | --- | --- | --- |
| **SL 1** | Basic | Casual observer | Individual | None | Accidental |
| **SL 2** | Enhanced | Hacker, insider | Low | Generic | Low |
| **SL 3** | High | Professional attacker, hacktivist | Moderate | System-specific | Moderate |
| **SL 4** | Critical | Nation-state, organized crime | Extended | System-specific | High |

### 3.2 Mapping to ISO 21434

IEC 62443 SL maps to ISO 21434 as follows:

| IEC 62443 SL | ISO 21434 Equivalent | Description |
| --- | --- | --- |
| SL 1 | CAL 1 | Basic cybersecurity controls |
| SL 2 | CAL 2 | Enhanced controls; independent review |
| SL 3 | CAL 3 | High controls; pen testing required |
| SL 4 | CAL 4 | Maximum controls; comprehensive testing |

---

## 4. SL-T by Zone

→ See `Zone_Conduit_Model.md` for detailed zone definitions.

| Zone | SL-T | Attacker Profile Addressed | Justification |
| --- | --- | --- | --- |
| Zone 1: Device Core | **SL 3** | Professional attacker with moderate resources, system-specific skills | Root of trust; compromise is persistent and irreversible. Must resist sophisticated attack. |
| Zone 2: Device Application | **SL 2** | Hacker with generic skills, low resources | Isolated from root of trust by Zone 1 boundary. Compromise contained. |
| Zone 3: Backend Infrastructure | **SL 3** | Professional attacker with moderate resources | Controls fleet-scale firmware distribution. High-value target. |
| Zone 4: Manufacturing | **SL 2** | Insider with low resources | Physically secured + time-limited exposure. Enhanced controls sufficient. |
| Zone 5: Network | **SL 2** | Hacker with generic skills | Untrusted by design. End-to-end protection at application layer. |
| Zone 6: Toolchain | **SL 3** | Professional attacker, potential nation-state | Supply chain compromise affects entire fleet. Must resist sophisticated attack. |

---

## 5. SL Achievement Plan

### 5.1 Per-Zone SL Gap Analysis

| Zone | SL-T | Current SL-A | Gap | Required Action |
| --- | --- | --- | --- | --- |
| Zone 1 | SL 3 | SL 2 | SL 2→3 | Hardware secure element for key protection, side-channel countermeasures |
| Zone 2 | SL 2 | SL 2 | — | None |
| Zone 3 | SL 3 | SL 2 | SL 2→3 | Formal backend security assessment, pen testing |
| Zone 4 | SL 2 | SL 1 | SL 1→2 | Manufacturing security specification (Phase 3.5) |
| Zone 5 | SL 2 | SL 1 | SL 1→2 | Transport security specification |
| Zone 6 | SL 3 | SL 1 | SL 1→3 | Supply chain security specification (Phase 3.1) |

### 5.2 Control-to-SL Mapping

Each security control contributes to achieving the SL-T for its zone:

| Control Category | SL 1 | SL 2 | SL 3 | SL 4 |
| --- | --- | --- | --- | --- |
| Signature Verification | Required | Required | Required | Required |
| Hash Verification | Required | Required | Required | Required |
| Version Enforcement | — | Required | Required | Required |
| Key Isolation | — | Required | Required (HW) | Required (HW + active shield) |
| Secure Debug | — | Required | Required | Required |
| Side-Channel Resistance | — | — | Required | Required |
| Physical Tamper Detection | — | — | Required | Required (active response) |
| Supply Chain Integrity | — | — | Required | Required |
| Formal Verification | — | — | Recommended | Required |
| Penetration Testing | — | — | Required | Required |

---

## 6. SL-C (Capability) by SBOP Component

SL-C defines the inherent security capability of each SBOP component — the maximum SL it can support if properly deployed and configured.

| Component | SL-C | Limiting Factor |
| --- | --- | --- |
| Boot Manager | SL 4 | State machine design is inherently capable of SL 4 |
| Crypto Engine | SL 3 | SL 4 requires post-quantum crypto (future) |
| Identity Manager | SL 3 | SL 4 requires hardware immutable identity |
| OTA Manager | SL 3 | SL 4 requires formal protocol verification |
| Backend | SL 3 | SL 4 requires fully air-gapped signing infrastructure |
| Secure Debug | SL 3 | SL 4 requires tamper-responsive debug disable |

---

## 7. Verification Requirements by CAL

### 7.1 Testing Requirements

| Activity | CAL 1 | CAL 2 | CAL 3 | CAL 4 |
| --- | --- | --- | --- | --- |
| Unit testing | Required | Required | Required | Required |
| Integration testing | Optional | Required | Required | Required |
| Negative testing | Optional | Recommended | Required | Required |
| Fuzz testing | Optional | Recommended | Required | Required |
| Fault injection | — | Optional | Required | Required |
| Penetration testing | — | — | Required | Required |
| Side-channel analysis | — | — | Required | Required |
| Formal verification | — | Optional | Recommended | Required |

### 7.2 SBOP Test Coverage

| Test Activity | SBOP Document | CAL Supported |
| --- | --- | --- |
| Unit testing | `Test_Cases.md` (TEST-BOOT, TEST-CRYPTO) | CAL 1 |
| Negative testing | `Test_Cases.md` (Section 6) | CAL 2 |
| Fuzz testing | `Fuzzing_Strategy.md` | CAL 3 |
| Fault injection | `Fault_Injection_Test.md` | CAL 3 |
| Penetration testing | `Red_Team_Test_Plan.md` | CAL 3 |
| Side-channel | `Side_Channel_Countermeasures.md` (Phase 3.3) | CAL 3 |

---

## 8. Compliance Evidence Matrix

### 8.1 Evidence by CAL Level

| CAL Level | Required Evidence | SBOP Artifact |
| --- | --- | --- |
| CAL 1 | Design documents, unit test results | `02_System_Design/`, `Test_Cases.md` |
| CAL 2 | + Independent review report, integration test results | `Review_and_Audit_Process.md`, traceability |
| CAL 3 | + Penetration test report, fault injection results, side-channel analysis | `Red_Team_Test_Plan.md`, `Fault_Injection_Test.md`, `Side_Channel_Countermeasures.md` |
| CAL 4 | + Comprehensive test report, formal verification proof | `Formal_Verification.md` |

### 8.2 Audit Readiness

For each CAL level, the following must be available for auditor review:

1. Complete requirements traceability (→ `Traceability_Matrix.md`)
2. Design documents with review history (→ `Security_Case.md`)
3. Test plans and results (→ `05_Verification/`)
4. Risk assessment with treatment decisions (→ `TARA_Methodology.md`)
5. Security incident response records (→ `Incident_Response.md`)
