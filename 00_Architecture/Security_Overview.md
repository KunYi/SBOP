# SBOP Security Overview

**Document ID:** SEC-OV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Security Objectives

| Objective | Goal | Key Mechanism |
| --- | --- | --- |
| Firmware authenticity | G-AUTH | Ed25519 signature verification |
| Firmware integrity | G-INT | SHA-256 hash verification, constant-time compare |
| Anti-rollback | (subset of G-AUTH) | Monotonic OTP version counter |
| Device identity | G-CLONE | TRNG UID + HKDF-derived KD, backend dedup |
| Key confidentiality | G-CONF | Secure element / TEE, KeyRef handles, zeroization |
| Availability | G-AVAIL | Dual-image A/B, OTA rollback, FAILSAFE recovery |
| Supply chain provenance | G-SC | Reproducible builds, SBOM, HSM signing, commit signing |
| Physical tamper resistance | G-PHY | Tamper sensors, glitch detection, redundant checks |
| Side-channel resistance | G-SCH | Constant-time crypto, masking, TVLA |
| Debug security | G-DBG | Debug lifecycle, challenge-response auth, rate limiting |
| Manufacturing integrity | G-MFR | Encrypted provisioning, batch authorization, backend dedup |

→ See `../04_Security/Security_Goals.md` for the complete goal trace.

---

## 2. Threat Model (Summary)

**Attacker capabilities considered:**
- Remote network attacker (all SL)
- Local physical attacker with debug tools (SL 2+)
- Fault injection (voltage/clock glitch) (SL 3+)
- Side-channel analysis (power/EM/timing) (SL 3+)
- Chip decapsulation and microprobing (SL 4)
- Supply chain interdiction (SL 4)
- Malicious insider (all SL, procedural controls)

→ See `Threat_Model.md` for detailed actor profiles and capabilities.
→ See `../04_Security/Attack_Tree.md` for 53 attack leaf nodes across 11 root branches.

**Out of scope for SL ≤ 3:**
- Advanced invasive hardware attacks (FIB, microprobing) — accepted residual risk
- Silicon-level backdoors (vendor test interfaces) — accepted residual risk

---

## 3. Security Principles

| Principle | Implementation |
| --- | --- |
| Zero trust | Every input verified; no implicit trust across boundaries |
| Least privilege | Zone 1/2 separation; MPU enforcement; key access scoped by context |
| Defense in depth | OTA verification + boot verification; redundant checks for fault injection |
| Cryptographic enforcement | No policy-based bypass; security enforced by cryptographic verification |
| Fail-safe | All verification failures → FAILSAFE; no recovery without re-verification |
| Secure by default | All security features enabled; no opt-out for critical controls |
| Open design | Security does not rely on secrecy of algorithms or specification |

---

## 4. Security Domains

| Domain | Scope Document | Requirements | Tests |
| --- | --- | --- | --- |
| Boot Security | `Boot_Flow_Pseudocode.md`, `Boot_State_Detail.md` | REQ-FR-BOOT-001..007 | TEST-BOOT-001..005 |
| Update Security | `OTA_Flow_Pseudocode.md` | REQ-FR-OTA-001..007 | TEST-OTA-001..004 |
| Identity Security | `Provisioning_Flow.md`, `Anti_Cloning.md` | REQ-SR-ID-001..002 | TEST-ID-001..003 |
| Crypto Security | `Crypto_Algorithms.md`, `Side_Channel_Countermeasures.md` | REQ-SR-SCH-001 | TIM-001/002, TVLA |
| Backend Security | `Key_Management.md`, `OTA_Deployment.md` | REQ-SR-OTA-001..003 | Backend audit |
| Supply Chain Security | `Supply_Chain_Security.md` | REQ-SR-SC-001..002 | SC-AUDIT-001..004 |
| Physical Security | `Physical_Tamper_Resistance.md` | REQ-SR-PHY-001..002 | FI-001..004, Physical pentest |
| Side-Channel | `Side_Channel_Countermeasures.md` | REQ-SR-SCH-001 | TIM-001/002, TVLA |
| Debug Security | `Secure_Debug_Architecture.md` | REQ-SR-DBG-001..002 | DBG-TEST-001..002 |
| Manufacturing Security | `Manufacturing_Security.md` | REQ-SR-MFR-001..002 | Factory audit |

---

## 5. Control Strategy Summary

| Control Layer | Mechanism |
| --- | --- |
| Preventive | Signature verification, hash verification, anti-rollback, debug lock, MPU isolation |
| Detective | Tamper sensors, glitch counters, backend anomaly detection, CVE monitoring |
| Corrective | OTA rollback, FAILSAFE recovery, key zeroization, incident response |
| Procedural | HSM quorum, signing ceremony, code review, audit log review |

→ See `../04_Security/Security_Controls.md` for 14 preventive + 8 detective + 7 corrective controls.

---

## 6. Standards and Compliance

→ See `../07_Compliance/Compliance_Overview.md` for full standards coverage:

| Standard | Domain | Detailed Mapping |
| --- | --- | --- |
| ISO/SAE 21434 | Automotive | `ISO_21434_Detailed_Mapping.md` |
| IEC 62443-4-1/4-2 | Industrial | `IEC_62443_Detailed_Mapping.md` |
| ECSS-E-ST-40 / Q-ST-80 | Space | `ECSS_Compliance_Mapping.md` |
| IEC 62304 / ISO 14971 | Medical | `Medical_Compliance_Mapping.md` |
| DO-178C / DO-254 / DO-326A | Aviation | `Aviation_Compliance_Mapping.md` |
| IEC 61508 / ISO 26262 | Functional Safety | `../04_Security/Safety_Analysis.md` |

---

## 7. References

| Document | Reference |
| --- | --- |
| Threat Model | `Threat_Model.md` |
| Security Goals | `../04_Security/Security_Goals.md` |
| Attack Tree | `../04_Security/Attack_Tree.md` |
| Security Controls | `../04_Security/Security_Controls.md` |
| Architecture Decision Record | `Architecture_Decision_Record.md` |
