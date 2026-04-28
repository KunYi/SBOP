# Traceability Matrix

**Document ID:** VER-TM-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Links requirements to design and verification artifacts. This is the master traceability document — all requirements must be mapped, no orphans allowed.

---

## 2. Traceability Structure

```
REQ → Design → Subsystem → Test → Evidence
```

---

## 3. Functional Requirements

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-FR-BOOT-001 | Boot_Overview, Trust_Model | Boot | TEST-BOOT-001, TEST-BOOT-002 |
| REQ-FR-BOOT-002 | Boot_Flow_Pseudocode | Boot | TEST-BOOT-001 |
| REQ-FR-BOOT-003 | Boot_Overview, Image_Format | Boot | TEST-BOOT-003 |
| REQ-FR-BOOT-004 | Boot_State_Detail, Key_Hierarchy | Boot | TEST-BOOT-004 |
| REQ-FR-BOOT-005 | Boot_Failure_Model | Boot | TEST-BOOT-005 |
| REQ-FR-BOOT-006 | Boot_Flow_Pseudocode | Boot | FI-001, FI-002 |
| REQ-FR-BOOT-007 | Boot_State_Detail, Trust_Model | Boot | TEST-BOOT-005, TEST-BOOT-001 |
| REQ-FR-OTA-001 | OTA_Overview, OTA_Flow_Pseudocode | Update | TEST-OTA-001 |
| REQ-FR-OTA-002 | OTA_Flow_Pseudocode | Update | TEST-OTA-002 |
| REQ-FR-OTA-003 | OTA_Failure_Model | Update | TEST-OTA-003 |
| REQ-FR-OTA-004 | OTA_Recovery | Update | TEST-OTA-004 |
| REQ-FR-OTA-005 | OTA_Flow_Pseudocode | Update | TEST-OTA-002 |
| REQ-FR-OTA-006 | OTA_Failure_Model | Update | TEST-OTA-003 |
| REQ-FR-OTA-007 | OTA_Overview | Update | TEST-OTA-001 |

## 4. Security Requirements (Identity)

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-SR-ID-001 | Identity_Overview, Provisioning_Flow | Identity | TEST-ID-001 |
| REQ-SR-ID-002 | Anti_Cloning, Identity_Overview | Identity | TEST-ID-002 |
| REQ-SR-ID-003 | Identity_Failure_Model | Identity | TEST-ID-003 |

## 5. Security Requirements (Supply Chain — A6)

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-SR-SC-001 | Supply_Chain_Security | Toolchain | SC-AUDIT-001 |
| REQ-SR-SC-002 | Supply_Chain_Security | Build Pipeline | SC-AUDIT-002 |
| REQ-SR-SC-003 | Supply_Chain_Security | HSM Operations | SC-AUDIT-003 |
| REQ-SR-SC-004 | Supply_Chain_Security | Dependency Mgmt | SC-AUDIT-004 |

## 6. Security Requirements (Physical Tamper — A7)

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-SR-PHY-001 | Physical_Tamper_Resistance | Tamper Detection | FI-001, Physical pentest |
| REQ-SR-PHY-002 | Physical_Tamper_Resistance | Key Zeroization | FI-002, FI-003, FI-004, Physical pentest |
| REQ-SR-PHY-003 | Physical_Tamper_Resistance | Secure Element | Physical pentest |

## 7. Security Requirements (Side-Channel — A8)

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-SR-SCH-001 | Side_Channel_Countermeasures | Crypto | TIM-001, TIM-002, TVLA |

## 8. Security Requirements (Debug — A9)

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-SR-DBG-001 | Secure_Debug_Architecture | Debug Lifecycle | DBG-TEST-001 |
| REQ-SR-DBG-002 | Secure_Debug_Architecture | Debug Auth | DBG-TEST-002 |

## 9. Security Requirements (Manufacturing — A10)

| Requirement | Design Reference | Subsystem | Verification |
|-------------|-----------------|-----------|--------------|
| REQ-SR-MFR-001 | Manufacturing_Security | Provisioning | Factory audit + TEST-ID-001..003 |
| REQ-SR-MFR-002 | Manufacturing_Security | Device Tracking | Factory audit + backend check |

## 10. Non-Functional Requirements

| Requirement | Design Reference | Verification |
|-------------|-----------------|--------------|
| REQ-NFR-SYS-001 | Boot_Flow_Pseudocode | Performance measurement (boot latency) |
| REQ-NFR-SYS-002 | OTA_Flow_Pseudocode | Performance measurement (OTA time) |
| REQ-NFR-SYS-003 | System_Decomposition | Resource measurement (RAM/Flash) |
| REQ-NFR-SYS-004 | Crypto_Algorithms | Security review, key strength audit |
| REQ-NFR-SYS-005 | Boot_Failure_Model, OTA_Failure_Model | Fault injection, reliability testing |
| REQ-NFR-SYS-006 | Compliance_Overview | Audit trace verification |
| REQ-NFR-SYS-007 | Security_Overview, Trust_Model | Security review |

## 11. Crypto Requirements

| Requirement | Design Reference | Verification |
|-------------|-----------------|--------------|
| REQ-SR-CRYPTO-001 | Crypto_Algorithms | NIST test vectors |
| REQ-SR-CRYPTO-002 | Key_Derivation | HKDF test vectors |
| REQ-SR-CRYPTO-003 | Signature_Format | Parsing test vectors |

## 12. Negative Tests

| Test ID | Target | Description |
|---------|--------|-------------|
| NEG-001 | Boot | Invalid signature (flipped bit) |
| NEG-002 | Boot | Corrupted hash |
| NEG-003 | Boot | Rollback version |
| NEG-004 | OTA | Truncated download |
| NEG-005 | OTA | Invalid TLS certificate |
| NEG-006 | Identity | Duplicate UID registration |
| NEG-007 | Crypto | Malformed signature DER |
| NEG-008 | Crypto | Non-constant-time comparison |

## 13. Requirements

- All REQs must be mapped — no orphan requirements
- No test without a requirement — every test traces to a REQ
- Every attack tree branch (A1-A11) must have requirements + tests
- Gaps in this matrix are blockers for release
