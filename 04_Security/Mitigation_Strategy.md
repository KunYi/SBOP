# Mitigation Strategy

**Document ID:** SEC-MIT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document maps every attack vector from the SBOP attack tree (→ `Attack_Tree.md`) to its corresponding mitigation, and links each mitigation to its design specification, implementing requirement, and verification test.

---

## 2. Mitigation Catalog

### 2.1 Unauthorized Firmware Execution (A1)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A1.1.1 Skip verification (fault injection) | Redundant verification + state machine enforcement | REQ-SR-BOOT-001 | Boot_State_Detail, Physical_Tamper_Resistance §7.2 | FI-001 |
| A1.1.2 Exploit parsing bug | Input validation, fuzzing, no fallback parsing | REQ-FR-BOOT-001 | Image_Format §11, Fuzzing_Strategy | Fuzz tests |
| A1.2.1 Break crypto | Standardized, reviewed algorithms (no custom crypto) | REQ-SR-BOOT-001 | Crypto_Algorithms §1-2 | Crypto review |
| A1.2.2 Steal signing key | HSM protection, multi-party quorum, air-gapped signing | REQ-SR-KEY-001 | Supply_Chain_Security §4, Key_Management | Signing audit |
| A1.3.1 Tamper during transfer | End-to-end signature verification on device | REQ-SR-OTA-001 | OTA_Flow_Pseudocode, Data_Flow | TEST-OTA-003 |
| A1.3.2 Backend compromise | Access control, audit logging, deployment policy | REQ-SR-OTA-001 | Key_Management, OTA_Deployment | Backend audit |

### 2.2 Firmware Integrity Compromise (A2)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A2.1 Modify after signing | SHA-256 hash verification on device | REQ-FR-BOOT-002 | Image_Format §9, Crypto_Algorithms §2 | TEST-BOOT-003 |
| A2.2 Hash verification bug | Standard SHA-256, no custom logic, fuzzing | REQ-FR-BOOT-002 | Crypto_Algorithms §2, Fuzzing_Strategy | Crypto tests |

### 2.3 Rollback Attack (A3)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A3.1 Install old firmware | Monotonic version counter in OTP/secure storage | REQ-FR-BOOT-004, REQ-SR-OTA-002 | Boot_State_Detail, Unified_State_Model | TEST-BOOT-004 |
| A3.2 Bypass version check | Redundant version check, fault injection defense | REQ-FR-BOOT-004 | Boot_Flow_Pseudocode (Phase 6), Fault_Injection_Test | FI-004 |

### 2.4 Device Cloning (A4)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A4.1 Extract keys | Key isolation in secure element / TEE | REQ-SR-KEY-001 | Crypto_Overview, Physical_Tamper_Resistance §3 | Analysis |
| A4.2 Copy identity data | Identity binding to hardware (PUF or HUK), backend validation | REQ-SR-ID-001, REQ-SR-ID-002 | Identity_Overview, Anti_Cloning | TEST-ID-002 |

### 2.5 OTA Exploitation (A5)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A5.1 Interrupt update | Dual-image model, atomic slot state transitions | REQ-FR-OTA-004, REQ-FR-OTA-005 | OTA_Flow_Pseudocode §4, OTA_Recovery | TEST-OTA-002 |
| A5.2 Corrupt storage | Slot metadata CRC, write-verify on install | REQ-FR-OTA-003 | OTA_Flow_Pseudocode (Phase 5), OTA_Failure_Model | TEST-OTA-003 |

### 2.6 Supply Chain Compromise (A6)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A6.1.1 Malicious commit (insider) | Commit signing, mandatory review, two-person rule | REQ-SR-SC-001 | Supply_Chain_Security §3.2 | SC-AUDIT-001 |
| A6.1.2 Compromised developer credentials | MFA enforcement, hardware token requirement, session timeouts | REQ-SR-SC-001 | Supply_Chain_Security §3.2 | SC-AUDIT-001 |
| A6.2.1 Build server compromise | Reproducible builds, ephemeral/isolated build environment | REQ-SR-SC-001 | Supply_Chain_Security §3.1 | SC-AUDIT-001 |
| A6.2.2 Malicious build tool/dependency | Dependency pinning, checksum verification, SBOM audit | REQ-SR-SC-001, REQ-SR-SC-002 | Supply_Chain_Security §3.3 | SC-AUDIT-002 |
| A6.3.1 Steal signing key from HSM | HSM-enforced access, quorum authorization, audit logging | REQ-SR-KEY-001 | Supply_Chain_Security §4.1 | SC-AUDIT-003 |
| A6.3.2 Sign unauthorized via insider | Multi-party signing, operator audit, signing ceremony | REQ-SR-KEY-001 | Supply_Chain_Security §4.2 | Signing audit |
| A6.4.1 Replace firmware on CDN | TLS + device-side signature verification + hash check | REQ-SR-SC-001 | Supply_Chain_Security §5 | SC-AUDIT-004 |
| A6.4.2 MITM on firmware download | TLS 1.3 with certificate pinning; device re-verifies signature and hash post-download | REQ-SR-OTA-001 | OTA_Flow_Pseudocode (Phase 2-3) | TEST-OTA-003 |

### 2.7 Physical Tampering (A7)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A7.1.1 Bus sniffing | Memory encryption, secure element for keys | REQ-SR-PHY-001 | Physical_Tamper_Resistance §3, §7.1 | Physical pentest |
| A7.1.2 Flash readout (chip-off) | Storage encryption (optional per flags), secure element | REQ-SR-PHY-001 | Physical_Tamper_Resistance §3 | Physical pentest |
| A7.2.1 Voltage glitching | Glitch detection monitors, redundant execution | REQ-SR-PHY-001, REQ-SR-PHY-002 | Physical_Tamper_Resistance §7.2 | FI-001 |
| A7.2.2 Clock glitching | Clock monitor, RC watchdog, redundant checks | REQ-SR-PHY-001, REQ-SR-PHY-002 | Physical_Tamper_Resistance §7.2 | FI-001 |
| A7.2.3 EM fault injection | Shielding, redundant checks, glitch detection | REQ-SR-PHY-001 | Physical_Tamper_Resistance §2.2 | Physical pentest (SL 3+) |
| A7.3.1 Open enclosure | Tamper switch/es, active tamper response | REQ-SR-PHY-001, REQ-SR-PHY-002 | Physical_Tamper_Resistance §6 | Physical pentest |
| A7.3.2 Bypass tamper switch | Redundant tamper sensors (multiple switch types), continuous monitoring, active response on any trigger | REQ-SR-PHY-001, REQ-SR-PHY-002 | Physical_Tamper_Resistance §6.2 | Physical pentest |

### 2.8 Side-Channel Attack (A8)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A8.1.1 Timing oracle on signature verify | Constant-time ECDSA/Ed25519 verification | REQ-SR-SCH-001 | Side_Channel_Countermeasures §3 | TIM-002 |
| A8.1.2 Timing oracle on hash compare | `constant_time_compare` — no early return | REQ-SR-SCH-001 | Side_Channel_Countermeasures §3.2 | TIM-001 |
| A8.2.1 SPA on ECDSA verification | Fixed-sequence operations, balanced point ops | SCH-SPA-001 through SCH-SPA-003 | Side_Channel_Countermeasures §4.1 | TVLA (FvR) |
| A8.2.2 DPA on key derivation | Masking, blinding (SL 3+) | SCH-DPA-001, SCH-DPA-002 | Side_Channel_Countermeasures §4.2 | TVLA (FvR) |
| A8.3.1 EM leakage from crypto operations | EM shielding (PCB/package level), balanced routing, low-emission logic styles | REQ-SR-SCH-001 | Side_Channel_Countermeasures §5 | EM emanation test (SL 3+) |

### 2.9 Debug Port Exploitation (A9)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A9.1.1 Debug left open after provisioning | Debug lock as final provisioning step; fuse programming | REQ-SR-DBG-001 | Secure_Debug_Architecture §3.2 (Phase 3) | DBG-TEST-001 |
| A9.1.2 Debug auth bypass | Device-specific KD_Debug, rate limiting, permanent lock after 10 failures | REQ-SR-DBG-002 | Secure_Debug_Architecture §4 | DBG-TEST-002 |
| A9.2.1 Steal debug auth credentials | Station security audit; KD_Debug never stored on station | REQ-SR-DBG-002 | Secure_Debug_Architecture §4.3 | Station audit |
| A9.2.2 Brute-force debug auth challenge | Exponential backoff, permanent lock after N failures | REQ-SR-DBG-002 | Secure_Debug_Architecture §4.2 | DBG-TEST-002 |
| A9.3.1 Inject breakpoint to skip verification | Debug disabled during boot (if LOCKED) | REQ-SR-DBG-001 | Secure_Debug_Architecture §8.2 | Fault injection test |

### 2.10 Manufacturing Bypass (A10)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A10.1.1 Overproduction | Batch authorization, backend counts devices, audit | REQ-SR-MFR-002 | Manufacturing_Security §9.1-9.2 | Factory audit |
| A10.2.1 Clone UID at factory | UID from TRNG (per-device entropy), attestation proof | REQ-SR-MFR-001, REQ-SR-ID-002 | Manufacturing_Security §4.2 (Step 3-4) | Backend dedup check |
| A10.2.2 Copy identity between devices | Device attestation key unique per device; backend dedup | REQ-SR-ID-002 | Identity_Overview, Anti_Cloning | TEST-ID-002 |
| A10.3.1 Intercept key injection | Encrypted provisioning channel, key wrapping | REQ-SR-MFR-001 | Manufacturing_Security §4.2 (Step 7) | Crypto audit |
| A10.4.1 Ship with test/debug enabled | Lock verification at end of provisioning line | REQ-SR-DBG-001, REQ-SR-MFR-001 | Manufacturing_Security §6.2 (Step 7-8) | Fuse readback test |
| A10.4.2 Unauthorized firmware via test mode | Test firmware signed with test-specific key (separate from production) | REQ-SR-MFR-001 | Manufacturing_Security §6.1 | Hash verification |

### 2.11 Zone Boundary Violation (A11)

| Attack | Mitigation | Requirement | Design | Test |
| --- | --- | --- | --- | --- |
| A11.1.1 Application reads boot memory (Zone 2 → Zone 1) | MPU/MMU memory protection; Zone 1 locked read-only after boot | REQ-FR-BOOT-007 | Zone_Conduit_Model §7.1, Reference_Architecture §3.3 | Memory protection test |
| A11.1.2 OTA writes to active slot bypassing boot | Boot slot selection enforced in Zone 1; Zone 2 cannot modify | REQ-SR-BOOT-001 | Boot_Interface, OTA_Flow_Pseudocode (inactive-slot-only check) | Access control test |
| A11.2.1 Bypass TLS mutual auth | TLS 1.3 with client certificate (KD-derived) + server cert verification | REQ-SR-OTA-001 | OTA_Flow_Pseudocode (Phase 1) | TLS test |
| A11.2.2 Replay captured OTA traffic | Challenge-response with fresh nonce per auth session | REQ-SR-OTA-001 | OTA_Flow_Pseudocode (Phase 1) | Replay test |
| A11.3.1 Unauthorized provisioning station connects to backend | Station authentication, mutual TLS, station whitelist | REQ-SR-MFR-001 | Manufacturing_Security §4.2 (Step 2) | Station audit |
| A11.3.2 Station malware exfiltrates keys | Station network isolation, no internet access, key wrapping | REQ-SR-MFR-001 | Manufacturing_Security §3.2 | Station security audit |

---

## 3. Mitigation Hierarchy

| Priority | Approach | Application in SBOP |
| --- | --- | --- |
| 1. Eliminate | Remove the attack surface entirely | Debug port permanently disabled after provisioning |
| 2. Prevent | Make the attack technically infeasible | HSM prevents key extraction; constant-time prevents timing oracle |
| 3. Detect | Detect the attack in progress | Tamper sensors, glitch counters, backend anomaly detection |
| 4. Respond | Limit damage and enable recovery | Key zeroization on tamper, OTA rollback, device lockdown |
| 5. Recover | Return to a known-good state | Recovery boot, dual-image fallback, backend-forced update |

---

## 4. Coverage Summary

| Attack Root | Leaf Nodes | Mitigations Defined | Coverage |
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
