# Security Case

**Document ID:** SEC-CASE-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Provides a unified view of the system security argument, linking threats, risks, mitigations, and verification. This document is the primary reference for security audits.

---

## 2. Structure

This document serves as the single entry point for all security-related artifacts. All other security documents must be consistent with this document.

---

## 3. Security Argument Steps

| Step | Activity | Reference |
|------|----------|-----------|
| 1 | Threat Definition | `Threat_Model.md` |
| 2 | Attack Analysis | `Attack_Tree.md` |
| 3 | Risk Evaluation | `Risk_Quantification.md` |
| 4 | Mitigation Strategy | `Security_Controls.md` |
| 5 | Requirements Mapping | `Security_Trace.md`, `Enhanced_Security_Trace.md` |
| 6 | Verification | `../05_Verification/` |

---

## 4. Security Claims

### C1: Firmware Authenticity
**Claim:** Only authenticated firmware can execute on the device.
**Attack tree nodes:** A1 (Execute Unauthorized Firmware — covers signature forgery A1.1, key compromise A1.2.2, OTA injection A1.3)
**Enforced by:** Ed25519 signature verification in bootloader, KI public key in Zone 1 read-only
**Evidence:** TEST-BOOT-002, FI-001

### C2: Firmware Integrity
**Claim:** Firmware cannot be modified after signing without detection.
**Attack tree nodes:** A2 (Break Firmware Integrity)
**Enforced by:** SHA-256 hash verification, constant-time comparison
**Evidence:** TEST-BOOT-003, FI-002

### C3: Anti-Rollback
**Claim:** An attacker cannot install an older, vulnerable firmware version.
**Attack tree nodes:** A3 (Perform Rollback Attack)
**Enforced by:** Monotonic OTP version counter, version comparison in boot
**Evidence:** TEST-BOOT-004

### C4: Supply Chain Integrity
**Claim:** Firmware cannot be tampered with during build, signing, or distribution.
**Attack tree nodes:** A6 (Supply Chain)
**Enforced by:** Reproducible builds, SBOM, commit signing, HSM signing, device-side re-verification
**Evidence:** SC-AUDIT-001, SC-AUDIT-002, SC-AUDIT-003, SC-AUDIT-004

### C5: Physical Tamper Resistance
**Claim:** Physical attacks (glitching, probing) cannot bypass security checks.
**Attack tree nodes:** A7 (Physical Tamper)
**Enforced by:** Tamper sensors, glitch detection, redundant checks, secure element, MPU isolation
**Evidence:** FI-001, FI-002, FI-003, FI-004, Physical pentest report

### C6: Side-Channel Resistance
**Claim:** Cryptographic operations do not leak key material through timing, power, or EM.
**Attack tree nodes:** A8 (Side-Channel)
**Enforced by:** Constant-time algorithms, masking, hardware countermeasures (SL 3+)
**Evidence:** TIM-001, TIM-002, TVLA report

### C7: Secure Debug
**Claim:** Debug access does not expose key material or allow security bypass.
**Attack tree nodes:** A9 (Debug)
**Enforced by:** Debug lifecycle states, challenge-response authentication, rate limiting, KeyRef handles
**Evidence:** DBG-TEST-001, DBG-TEST-002

### C8: Manufacturing Security
**Claim:** Device identity cannot be cloned or compromised during manufacturing.
**Attack tree nodes:** A10 (Manufacturing)
**Enforced by:** Station attestation, encrypted provisioning channel, KD zeroization after injection, backend dedup
**Evidence:** Factory audit report, TEST-ID-001, TEST-ID-002, TEST-ID-003

### C9: Zone Isolation
**Claim:** Compromise of Zone 2 (application) cannot escalate to Zone 1 (boot/crypto/identity).
**Attack tree nodes:** A11 (Zone Hopping)
**Enforced by:** MPU/MMU configuration at LOCK_BOOT, Zone 1 memory locked read-only, no Zone 2 → Zone 1 access paths
**Evidence:** TEST-BOOT-005, System architecture review

### C10: Key Confidentiality
**Claim:** Key material (KR, KD, sub-keys) is never exposed outside secure element / HSM.
**Attack tree nodes:** A1.2.2 (Steal signing key), A6.3.1 (HSM key compromise), A7 (Physical Tampering), A8 (Side-Channel)
**Enforced by:** Secure element / TEE storage, KeyRef handles only, HSM for KR, zeroization on tamper
**Evidence:** FI-003, TIM-001, TIM-002, TVLA, Architecture review

### C11: Device Identity Uniqueness
**Claim:** Each device has a unique, unforgeable identity.
**Attack tree nodes:** A4 (Clone Device Identity), A10 (Manufacturing Bypass)
**Enforced by:** TRNG UID (128-bit), KD = HKDF(KR, UID) binding, backend dedup, PUF where available
**Evidence:** TEST-ID-001, TEST-ID-002, TEST-ID-003

---

## 5. Claims Traceability

| Claim | Attack Tree | Requirements | Primary Evidence |
|-------|-------------|-------------|------------------|
| C1: Authenticity | A1 | REQ-FR-BOOT-001, REQ-FR-BOOT-002 | TEST-BOOT-002, FI-001 |
| C2: Integrity | A2 | REQ-FR-BOOT-003 | TEST-BOOT-003, FI-002 |
| C3: Anti-Rollback | A3 | REQ-FR-BOOT-004 | TEST-BOOT-004 |
| C4: Supply Chain | A6 | REQ-SR-SC-001, REQ-SR-SC-002 | SC-AUDIT-001..004 |
| C5: Physical | A7 | REQ-SR-PHY-001, REQ-SR-PHY-002 | FI-001..004, Physical pentest |
| C6: Side-Channel | A8 | REQ-SR-SCH-001 | TIM-001, TIM-002, TVLA |
| C7: Debug | A9 | REQ-SR-DBG-001, REQ-SR-DBG-002 | DBG-TEST-001, DBG-TEST-002 |
| C8: Manufacturing | A10 | REQ-SR-MFR-001, REQ-SR-MFR-002 | Factory audit, TEST-ID-001..003 |
| C9: Zone Isolation | A11 | REQ-FR-BOOT-007 | TEST-BOOT-005 |
| C10: Key Confidentiality | A1, A6, A7, A8 | REQ-SR-CONF-001 | FI-003, TIM-001/002, TVLA |
| C11: Identity Uniqueness | A4, A10 | REQ-SR-ID-001, REQ-SR-ID-002 | TEST-ID-001..003 |

---

## 6. Consistency Rules

- All security artifacts must align with `Attack_Tree.md`
- Risk entries must correspond to attack nodes
- All mitigations must map to requirements
- All requirements must be verifiable
- `Enhanced_Security_Trace.md` is the authoritative source for risk evaluation and security completeness
- Any inconsistency between referenced documents shall be resolved in favor of `Enhanced_Security_Trace.md`

---

## 7. References

| Document | Reference |
|----------|-----------|
| Threat Model | `../00_Architecture/Threat_Model.md` |
| Attack Tree | `Attack_Tree.md` |
| Risk Quantification | `Risk_Quantification.md` |
| Security Controls | `Security_Controls.md` |
| Security Trace | `Security_Trace.md` |
| Enhanced Security Trace | `Enhanced_Security_Trace.md` |
| Test Strategy | `../05_Verification/Test_Strategy.md` |
