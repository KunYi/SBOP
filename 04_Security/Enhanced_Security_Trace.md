# Enhanced Security Trace

**Document ID:** SEC-EST-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Extends traceability to include risk evaluation. Every attack tree leaf node (→ `Attack_Tree.md`) is mapped through:
Threat → Attack → Risk → Mitigation → Requirement → Test.

---

## 2. Format

Each row traces one attack tree leaf node through the full chain:

| Attack | Risk (TARA / Internal) | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |

Risk values are shown as: TARA risk level (→ `TARA_Methodology.md`) / Internal risk score (→ `Risk_Quantification.md`).

---

## 3. Branch A1: Execute Unauthorized Firmware

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A1.1.1 Skip verification (fault injection) | High (8) / Critical (20) | Redundant verification + state machine enforcement | REQ-SR-BOOT-001 | FI-001 |
| A1.1.2 Exploit parsing bug | Medium (3) / — | Input validation, fuzzing, no fallback parsing | REQ-FR-BOOT-001 | Fuzz tests |
| A1.2.1 Break crypto | Low (0) / Low (5) | Standardized, reviewed algorithms (no custom crypto) | REQ-SR-BOOT-001 | Crypto review |
| A1.2.2 Steal signing key | Medium (4) / High (15) | HSM protection, multi-party quorum, air-gapped signing | REQ-SR-KEY-001 | Signing audit |
| A1.3.1 Tamper during transfer | High (8) / — | End-to-end signature verification on device | REQ-SR-OTA-001 | TEST-OTA-003 |
| A1.3.2 Backend compromise | Medium (4) / — | Access control, audit logging, deployment policy | REQ-SR-OTA-001 | Backend audit |

---

## 4. Branch A2: Break Firmware Integrity

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A2.1 Modify after signing | Medium (4) / — | SHA-256 hash verification on device | REQ-FR-BOOT-002 | TEST-BOOT-003 |
| A2.2 Hash verification bug | Low (0) / — | Standard SHA-256, no custom logic, fuzzing | REQ-FR-BOOT-002 | Crypto tests |

---

## 5. Branch A3: Perform Rollback Attack

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A3.1 Install old firmware | Critical (9) / Critical (16) | Monotonic version counter in OTP/secure storage | REQ-FR-BOOT-004 | TEST-BOOT-004 |
| A3.2 Bypass version check | High (6) / — | Redundant version check, fault injection defense | REQ-FR-BOOT-004 | FI-004 |

---

## 6. Branch A4: Clone Device Identity

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A4.1 Extract keys | Medium (4) / High (15) | Key isolation in secure element / TEE | REQ-SR-KEY-001 | Analysis |
| A4.2 Copy identity data | High (6) / — | Identity binding to hardware (PUF or HUK), backend validation | REQ-SR-ID-001, REQ-SR-ID-002 | TEST-ID-002 |

---

## 7. Branch A5: Exploit OTA Mechanism

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A5.1 Interrupt update | Critical (9) / High (15) | Dual-image model, atomic slot state transitions | REQ-FR-OTA-004, REQ-FR-OTA-005 | TEST-OTA-002 |
| A5.2 Corrupt storage | High (6) / High (12) | Slot metadata CRC, write-verify on install | REQ-FR-OTA-003 | TEST-OTA-003 |

---

## 8. Branch A6: Compromise Supply Chain

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A6.1.1 Malicious commit (insider) | — / High (10) | Commit signing, mandatory review, two-person rule | REQ-SR-SC-001 | SC-AUDIT-001 |
| A6.1.2 Compromised developer credentials | — / — | MFA enforcement, hardware token requirement, session timeouts | REQ-SR-SC-001 | SC-AUDIT-001 |
| A6.2.1 Build server compromise | — / High (15) | Reproducible builds, ephemeral/isolated build environment | REQ-SR-SC-001 | SC-AUDIT-001 |
| A6.2.2 Malicious build tool/dependency | — / — | Dependency pinning, checksum verification, SBOM audit | REQ-SR-SC-001, REQ-SR-SC-002 | SC-AUDIT-002 |
| A6.3.1 Steal signing key from HSM | — / Low (5) | HSM-enforced access, quorum authorization, audit logging | REQ-SR-KEY-001 | SC-AUDIT-003 |
| A6.3.2 Sign unauthorized via insider | — / High (10) | Multi-party signing, operator audit, signing ceremony | REQ-SR-KEY-001 | Signing audit |
| A6.4.1 Replace firmware on CDN | — / High (12) | TLS + device-side signature verification + hash check | REQ-SR-SC-001 | SC-AUDIT-004 |
| A6.4.2 MITM on firmware download | — / — | TLS 1.3 with certificate pinning; device re-verifies post-download | REQ-SR-OTA-001 | TEST-OTA-003 |

---

## 9. Branch A7: Physical Tampering

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A7.1.1 Bus sniffing | — / High (12) | Memory encryption, secure element for keys | REQ-SR-PHY-001 | Physical pentest |
| A7.1.2 Flash readout (chip-off) | — / High (12) | Storage encryption (optional per flags), secure element | REQ-SR-PHY-001 | Physical pentest |
| A7.2.1 Voltage glitching | — / High (10) | Glitch detection monitors, redundant execution | REQ-SR-PHY-001, REQ-SR-PHY-002 | FI-001 |
| A7.2.2 Clock glitching | — / High (10) | Clock monitor, RC watchdog, redundant checks | REQ-SR-PHY-001, REQ-SR-PHY-002 | FI-001 |
| A7.2.3 EM fault injection | — / — | Shielding, redundant checks, glitch detection | REQ-SR-PHY-001 | Physical pentest (SL 3+) |
| A7.3.1 Open enclosure | — / High (12) | Tamper switch/es, active tamper response | REQ-SR-PHY-001, REQ-SR-PHY-002 | Physical pentest |
| A7.3.2 Bypass tamper switch | — / — | Redundant tamper sensors, continuous monitoring, active response | REQ-SR-PHY-001, REQ-SR-PHY-002 | Physical pentest |

---

## 10. Branch A8: Side-Channel Attack

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A8.1.1 Timing oracle (signature verify) | — / High (12) | Constant-time Ed25519 verification | REQ-SR-SCH-001 | TIM-002 |
| A8.1.2 Timing oracle (hash compare) | — / High (9) | `constant_time_compare` — no early return | REQ-SR-SCH-001 | TIM-001 |
| A8.2.1 SPA on Ed25519 verification | — / High (8) | Fixed-sequence operations, balanced point ops | SCH-SPA-001..003 | TVLA (FvR) |
| A8.2.2 DPA on key derivation | — / High (8) | Masking, blinding (SL 3+) | SCH-DPA-001, SCH-DPA-002 | TVLA (FvR) |
| A8.3.1 EM leakage from crypto ops | — / — | EM shielding, balanced routing, low-emission logic | REQ-SR-SCH-001 | EM emanation test (SL 3+) |

---

## 11. Branch A9: Debug Port Exploitation

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A9.1.1 Debug left open after provisioning | — / High (10) | Debug lock as final provisioning step; fuse programming | REQ-SR-DBG-001 | DBG-TEST-001 |
| A9.1.2 Debug auth bypass | — / Low (5) | Device-specific KD_Debug, rate limiting, permanent lock after 10 failures | REQ-SR-DBG-002 | DBG-TEST-002 |
| A9.2.1 Steal debug auth credentials | — / — | Station security audit; KD_Debug never stored on station | REQ-SR-DBG-002 | Station audit |
| A9.2.2 Brute-force debug auth challenge | — / — | Exponential backoff, permanent lock after N failures | REQ-SR-DBG-002 | DBG-TEST-002 |
| A9.3.1 Inject breakpoint to skip verification | — / High (10) | Debug disabled during boot (if LOCKED) | REQ-SR-DBG-001 | Fault injection test |

---

## 12. Branch A10: Manufacturing Bypass

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A10.1.1 Overproduction | — / High (9) | Batch authorization, backend counts devices, audit | REQ-SR-MFR-002 | Factory audit |
| A10.2.1 Clone UID at factory | — / High (10) | UID from TRNG (per-device entropy), attestation proof | REQ-SR-MFR-001, REQ-SR-ID-002 | Backend dedup check |
| A10.2.2 Copy identity between devices | — / — | Device attestation key unique per device; backend dedup | REQ-SR-ID-002 | TEST-ID-002 |
| A10.3.1 Intercept key injection | — / High (10) | Encrypted provisioning channel, key wrapping | REQ-SR-MFR-001 | Crypto audit |
| A10.4.1 Ship with test/debug enabled | — / High (8) | Lock verification at end of provisioning line | REQ-SR-DBG-001, REQ-SR-MFR-001 | Fuse readback test |
| A10.4.2 Unauthorized firmware via test mode | — / — | Test firmware signed with test-specific key (separate from production) | REQ-SR-MFR-001 | Hash verification |

---

## 13. Branch A11: Zone Boundary Violation

| Attack | Risk | Mitigation | Requirement | Test |
| --- | --- | --- | --- | --- |
| A11.1.1 Application reads boot memory (Zone 2 → Zone 1) | — / High (15) | MPU/MMU memory protection; Zone 1 locked read-only after boot | REQ-FR-BOOT-007 | Memory protection test |
| A11.1.2 OTA writes to active slot bypassing boot | — / — | Boot slot selection enforced in Zone 1; Zone 2 cannot modify | REQ-SR-BOOT-001 | Access control test |
| A11.2.1 Bypass TLS mutual auth | — / High (15) | TLS 1.3 with client certificate (KD-derived) + server cert verification | REQ-SR-OTA-001 | TLS test |
| A11.2.2 Replay captured OTA traffic | — / — | Challenge-response with fresh nonce per auth session | REQ-SR-OTA-001 | Replay test |
| A11.3.1 Unauthorized provisioning station connects to backend | — / — | Station authentication, mutual TLS, station whitelist | REQ-SR-MFR-001 | Station audit |
| A11.3.2 Station malware exfiltrates keys | — / High (10) | Station network isolation, no internet access, key wrapping | REQ-SR-MFR-001 | Station security audit |

---

## 14. Coverage Summary

| Attack Branch | Leaf Nodes | Traced | Coverage |
| --- | --- | --- | --- |
| A1: Unauthorized Firmware | 6 | 6 | 100% |
| A2: Firmware Integrity | 2 | 2 | 100% |
| A3: Rollback Attack | 2 | 2 | 100% |
| A4: Device Cloning | 2 | 2 | 100% |
| A5: OTA Exploitation | 2 | 2 | 100% |
| A6: Supply Chain Compromise | 8 | 8 | 100% |
| A7: Physical Tampering | 7 | 7 | 100% |
| A8: Side-Channel Attack | 5 | 5 | 100% |
| A9: Debug Exploitation | 5 | 5 | 100% |
| A10: Manufacturing Bypass | 6 | 6 | 100% |
| A11: Zone Boundary Violation | 6 | 6 | 100% |
| **Total** | **51** | **51** | **100%** |

---

## 15. Rules

* All attack tree leaf nodes MUST appear in this document
* Risk values: TARA rating (→ `TARA_Methodology.md`) is the compliance-auditable value; internal score (→ `Risk_Quantification.md`) is for engineering decisions
* "—" indicates the risk value has not been formally rated in the referenced document
* Each mitigation MUST reference at least one requirement
* Each requirement MUST reference at least one test or analysis
* Detailed mitigations → `Mitigation_Strategy.md`
