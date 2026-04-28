# Audit Traceability

**Document ID:** CMP-AT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines traceability from requirement to evidence for all security domains (A1-A11) and all 6 compliance standards.

---

## 2. Trace Structure

```
REQ → Design → Implementation → Test → Evidence
```

All requirements must have a complete trace chain. No broken links allowed. Trace must be auditable.

---

## 3. Boot & OTA Trace Chains (A1-A5)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-FR-BOOT-001 | Boot_Overview, Trust_Model | Signature verification in bootloader | TEST-BOOT-002 | Test report TR-BOOT-002 |
| REQ-FR-BOOT-002 | Boot_Flow_Pseudocode | Boot state machine | TEST-BOOT-001 | Test report TR-BOOT-001 |
| REQ-FR-BOOT-003 | Boot_Overview, Image_Format | SHA-256 integrity check | TEST-BOOT-003 | Test report TR-BOOT-003 |
| REQ-FR-BOOT-004 | Boot_State_Detail, Key_Hierarchy | OTP version counter | TEST-BOOT-004 | Test report TR-BOOT-004 |
| REQ-FR-BOOT-005 | Boot_Failure_Model | FAILSAFE handler | TEST-BOOT-005 | Test report TR-BOOT-005 |
| REQ-FR-BOOT-006 | Boot_Flow_Pseudocode | Redundant verification (SL 3+) | FI-001, FI-002 | FI report |
| REQ-FR-BOOT-007 | Boot_State_Detail | MPU/MMU lockdown | TEST-BOOT-005 | Test report |
| REQ-FR-OTA-001 | OTA_Overview, OTA_Flow_Pseudocode | OTA state machine | TEST-OTA-001 | Test report TR-OTA-001 |
| REQ-FR-OTA-002 | OTA_Flow_Pseudocode | Image verification before staging | TEST-OTA-002 | Test report TR-OTA-002 |
| REQ-FR-OTA-003 | OTA_Failure_Model | Rollback manager | TEST-OTA-003 | Test report TR-OTA-003 |
| REQ-FR-OTA-004 | OTA_Recovery | Recovery handler | TEST-OTA-004 | Test report TR-OTA-004 |
| REQ-FR-OTA-005 | OTA_Flow_Pseudocode | Dual-slot isolation | TEST-OTA-002 | Test report |
| REQ-FR-OTA-006 | OTA_Failure_Model | Automatic rollback logic | TEST-OTA-003 | Test report |
| REQ-FR-OTA-007 | OTA_Overview | mTLS client certificate | TEST-OTA-001 | Test report |

---

## 4. Identity Trace Chains (A5)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-SR-ID-001 | Identity_Overview, Provisioning_Flow | UID generation, KD injection | TEST-ID-001 | Test report TR-ID-001 |
| REQ-SR-ID-002 | Anti_Cloning, Identity_Overview | Backend dedup, hardware binding | TEST-ID-002 | Test report TR-ID-002 |
| REQ-SR-ID-003 | Identity_Failure_Model | Key zeroization | TEST-ID-003 | Test report TR-ID-003 |

---

## 5. Supply Chain Trace Chains (A6)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-SR-SC-001 | Supply_Chain_Security | Reproducible builds, SBOM generation | SC-AUDIT-001 | Build provenance + SBOM |
| REQ-SR-SC-002 | Supply_Chain_Security | Commit signing, branch protection | SC-AUDIT-002 | Repo configuration audit |
| REQ-SR-SC-003 | Supply_Chain_Security | HSM signing ceremony | SC-AUDIT-003 | Signing ceremony records |
| REQ-SR-SC-004 | Supply_Chain_Security | Dependency vulnerability monitoring | SC-AUDIT-004 | CVE scan report |

---

## 6. Physical Tamper Trace Chains (A7)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-SR-PHY-001 | Physical_Tamper_Resistance | Tamper detection sensors | FI-001 | Glitch test report |
| REQ-SR-PHY-002 | Physical_Tamper_Resistance | Redundant security checks | FI-002, FI-003, FI-004 | FI test reports |
| REQ-SR-PHY-003 | Physical_Tamper_Resistance | Secure element tamper response | Physical pentest | Pentest report |

---

## 7. Side-Channel Trace Chains (A8)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-SR-SCH-001 | Side_Channel_Countermeasures | Constant-time crypto, masking | TIM-001, TIM-002, TVLA | Timing/leakage reports |

---

## 8. Debug Security Trace Chains (A9)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-SR-DBG-001 | Secure_Debug_Architecture | Debug lifecycle state machine | DBG-TEST-001 | Debug lock verification |
| REQ-SR-DBG-002 | Secure_Debug_Architecture | Challenge-response auth, rate limiting | DBG-TEST-002 | Auth bypass test report |

---

## 9. Manufacturing Security Trace Chains (A10)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-SR-MFR-001 | Manufacturing_Security | Station attestation, encrypted provisioning | Factory audit | Audit report |
| REQ-SR-MFR-002 | Manufacturing_Security | KD zeroization after injection | TEST-ID-003 | Test report |

---

## 10. Zone Isolation Trace Chains (A11)

| Requirement | Design | Implementation | Test | Evidence |
|-------------|--------|----------------|------|----------|
| REQ-FR-BOOT-007 | Boot_State_Detail, Trust_Model | MPU/MMU configuration at LOCK_BOOT | TEST-BOOT-005 | Architecture review + test |

---

## 11. Standards-Specific Audit Traces

| Standard | Clause | Requirement | Evidence |
|----------|--------|-------------|----------|
| ISO 21434 | 9.5 (TARA) | Risk assessment | TARA_Methodology.md + Risk_Quantification.md |
| ISO 21434 | 10 (Concept) | Security concept | Security_Case.md + Attack_Tree.md |
| ISO 21434 | 11 (Development) | Implementation verification | TEST-BOOT-001..005, TEST-OTA-001..004 |
| ISO 21434 | 12 (Production) | Manufacturing controls | Manufacturing_Security.md + Factory audit |
| IEC 62443 | CR 3.2 (Malicious code) | REQ-FR-BOOT-001 | TEST-BOOT-002 |
| IEC 62443 | CR 4.2 (Least privilege) | Trust_Model.md, Zone_Conduit_Model.md | Architecture review |
| IEC 62443 | CR 5.2 (Integrity) | REQ-FR-BOOT-003 | TEST-BOOT-003 |
| IEC 62443 | CR 6.2 (Authentication) | REQ-SR-ID-001, KD_Auth mTLS | TEST-ID-001, TEST-OTA-001 |
| IEC 62443 | CR 7.2 (Resource) | REQ-NFR-SYS-003 | Performance test |
| ECSS | Q-ST-80 5.4.2 (SW requirements) | Requirements_Overview | Review records |
| ECSS | E-ST-40 5.4 (SW design) | System_Decomposition, State_Machine | Design review |
| ECSS | E-ST-70-41 (Telemetry) | Telemetry flow in System_Context | Integration test |
| IEC 62304 | 5.3.5 (Segregation) | System_Decomposition, Zone_Conduit_Model | Review + test |
| IEC 62304 | 5.7 (Risk management) | Risk_Quantification, Security_Case | Risk assessment review |
| ISO 14971 | 4.4 (Risk estimation) | Risk_Quantification | Risk matrix |
| ISO 14971 | 6.2 (Risk control) | Security_Controls | Control verification |
| DO-178C | MC/DC (Level A) | Boot verification logic | TEST-BOOT-001..005 |
| DO-178C | Traceability (Level A) | This document | Audit |
| DO-254 | Hardware assurance | Physical_Tamper_Resistance | FI-001..004 |
| DO-326A | Security process | Security_Case | Process audit |
| IEC 61508 | SIL 3 verification | All test artifacts | Test reports |

---

## 12. Requirements

- All REQs must have a complete trace chain (REQ → Design → Implementation → Test → Evidence)
- No broken links allowed
- Trace must be auditable — every link verifiable from both directions
- Coverage: all 11 attack tree branches (A1-A11) must have trace entries
