# Manufacturing Security

**Document ID:** SEC-MFR-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the security requirements and controls for the SBOP manufacturing and provisioning process. It covers factory security, secure key injection, device provisioning, manufacturing test, and rejected device handling.

---

## 2. Manufacturing Threat Model

### 2.1 Threat Actors

| Actor | Motivation | Capability |
| --- | --- | --- |
| Factory operator (insider) | Financial gain, sabotage | Physical access to provisioning equipment and devices |
| Factory visitor / contractor | Espionage, curiosity | Limited physical access |
| Supply chain intermediary | Cloning, overproduction | Access to devices between factory and customer |
| External attacker | Malicious firmware injection | May target provisioning network |
| Compromised provisioning station | Software compromise | Can inject unauthorized firmware or keys |

### 2.2 Threat Scenarios

| Threat ID | Scenario | Impact |
| --- | --- | --- |
| MFR-T-001 | **Overproduction**: Factory produces more devices than ordered | Unauthorized devices in field |
| MFR-T-002 | **Identity cloning**: Factory provisions multiple devices with same identity | Duplicate devices; backend confusion |
| MFR-T-003 | **Key extraction**: Attacker extracts keys during provisioning | Compromised device keys; impersonation |
| MFR-T-004 | **Unauthorized firmware**: Non-production firmware loaded at factory | Backdoored or debug-enabled devices shipped |
| MFR-T-005 | **Non-provisioned devices**: Devices shipped without provisioning (or partial provisioning) | Devices operate without backend registration; untrackable |
| MFR-T-006 | **Test mode persistence**: Test/debug modes not properly disabled | Devices ship with debug capabilities enabled |
| MFR-T-007 | **Provisioning data leak**: Provisioning logs or key material exposed | Mass compromise potential |
| MFR-T-008 | **Counterfeit device insertion**: Unauthorized device inserted into supply chain | Devices not bound to legitimate backend |

---

## 3. Factory Security Requirements

### 3.1 Physical Security

| Requirement | Description |
| --- | --- |
| MFR-PHY-001 | Provisioning area must be physically secured (access-controlled room/cage) |
| MFR-PHY-002 | Operator access to provisioning area must be authenticated (badge + PIN/biometric) |
| MFR-PHY-003 | Provisioning equipment must be within line-of-sight of security cameras |
| MFR-PHY-004 | Unauthorized entry to provisioning area must trigger alarm and halt provisioning |

### 3.2 Network Security

| Requirement | Description |
| --- | --- |
| MFR-NET-001 | Provisioning station must be on isolated network (no internet access) |
| MFR-NET-002 | Provisioning station must only communicate with backend over authenticated, encrypted channel |
| MFR-NET-003 | No wireless interfaces on provisioning station |
| MFR-NET-004 | Provisioning network traffic must be logged |

### 3.3 Operator Security

| Requirement | Description |
| --- | --- |
| MFR-OP-001 | All provisioning operations must be attributable to a specific operator |
| MFR-OP-002 | Operator authentication required before each provisioning session |
| MFR-OP-003 | Two-person rule for critical operations (root key handling, batch authorization) |
| MFR-OP-004 | Operator audit log must be immutable and regularly reviewed |

---

## 4. Secure Provisioning Protocol

### 4.1 Provisioning Flow

```
Provisioning Station                          Device                  Backend
       │                                        │                       │
       │  1. Establish physical connection      │                       │
       │─────────────────────────────────────> │                       │
       │                                        │                       │
       │  2. Mutual authentication              │                       │
       │<─────────────────────────────────────> │                       │
       │                                        │                       │
       │  3. Request device UID generation      │                       │
       │─────────────────────────────────────> │                       │
       │  4. UID + attestation proof            │                       │
       │<─────────────────────────────────────  │                       │
       │                                        │                       │
       │  5. Request device registration        │                       │
       │─────────────────────────────────────────────────────────────>│
       │                                        │                       │
       │  6. Registration confirmation + KD     │                       │
       │<─────────────────────────────────────────────────────────────│
       │                                        │                       │
       │  7. Inject KD                          │                       │
       │─────────────────────────────────────> │                       │
       │                                        │                       │
       │  8. Verify KD derivation               │                       │
       │<─────────────────────────────────────  │                       │
       │                                        │                       │
       │  9. Lock device                        │                       │
       │─────────────────────────────────────> │                       │
       │                                        │                       │
       │  10. Lock confirmation                 │                       │
       │<─────────────────────────────────────  │                       │
       │                                        │                       │
       │  11. Store provisioning record         │                       │
       │─────────────────────────────────────────────────────────────>│
```

### 4.2 Protocol Requirements

| Step | Requirement | Verification |
| --- | --- | --- |
| 1 | Physical connection must be dedicated (point-to-point; no shared bus) | Physical inspection |
| 2 | Mutual authentication using provisioned station key and device attestation key | Cryptographic verification |
| 3 | UID generation must use device TRNG | RNG quality test |
| 4 | Attestation proof: signature(device_attestation_key, UID || device_metadata) | Signature verification at station |
| 5 | Registration request includes UID, attestation proof, station operator ID, timestamp | Backend validation |
| 6 | Backend derives KD from KR + UID; returns to station over encrypted channel | TLS + application encryption |
| 7 | KD injection must be encrypted (key wrapping with device-specific transport key) | Encrypted channel verification |
| 8 | Device computes expected KD and compares; mismatch → abort provisioning | Constant-time comparison |
| 9 | Lock command triggers: debug disable, fuse programming, state transition LOCKED | Fuse readback verification |
| 10 | Lock confirmation: device responds with signed lock attestation | Signature verification |

---

## 5. Device Lifecycle Tracking

### 5.1 Provisioning Record

Each device must have a permanent provisioning record:

| Field | Description |
| --- | --- |
| Device UID | Unique device identifier |
| Device public key | Public key corresponding to attestation key |
| Provisioning timestamp | UTC timestamp of provisioning completion |
| Provisioning station ID | Unique identifier of provisioning station |
| Operator ID(s) | Operators who performed/authorized provisioning |
| Firmware version | Version of bootloader/firmware provisioned |
| Firmware hash | SHA-256 of provisioned firmware |
| Backend registration ID | Registration token from backend |
| Lock attestation | Signed confirmation of successful lock |

### 5.2 Backend Registration

After provisioning, the device is registered with the backend. The backend record must include:

- Full provisioning record
- Device operational state (OPERATIONAL / LOCKED / REVOKED)
- Current firmware version
- Update history log

---

## 6. Manufacturing Test Security

### 6.1 Test Mode Requirements

| Requirement | Description |
| --- | --- |
| MFR-TEST-001 | Manufacturing test must use test-specific firmware or test mode flag |
| MFR-TEST-002 | Test firmware must be signed with test-specific key (separate from production key) |
| MFR-TEST-003 | Test mode must not be accessible after provisioning lock |
| MFR-TEST-004 | Test firmware must not contain production signing keys |
| MFR-TEST-005 | Test artifacts (logs, debug data) must be cleared before provisioning lock |

### 6.2 Test-to-Production Transition

```
Step 1: Complete all manufacturing tests
Step 2: Verify test results pass all criteria
Step 3: Erase test firmware and test data
Step 4: Flash production bootloader
Step 5: Verify production bootloader hash
Step 6: Execute provisioning flow (Section 4)
Step 7: Lock device
Step 8: Verify lock state
Step 9: Package and ship
```

---

## 7. Rejected and Defective Device Handling

### 7.1 Rejection Categories

| Category | Cause | Handling |
| --- | --- | --- |
| R1: Test failure | Device fails manufacturing test | Re-test; if persistent failure → R3 |
| R2: Provisioning failure | Provisioning process interrupted or failed | Re-start provisioning from Step 1 |
| R3: Unrecoverable defect | Hardware defect or persistent provisioning failure | Secure decommissioning |
| R4: Suspected tamper | Tamper evidence during manufacturing | Quarantine; forensic analysis; decommission |

### 7.2 Secure Decommissioning

| Requirement | Description |
| --- | --- |
| MFR-DECOM-001 | All key material must be zeroized before device leaves factory |
| MFR-DECOM-002 | Backend registration must be revoked (if registered) |
| MFR-DECOM-003 | Decommissioned device UID must be added to blocklist |
| MFR-DECOM-004 | Physical destruction of secure element / key storage recommended |
| MFR-DECOM-005 | Decommissioning must be logged with operator attribution |

---

## 8. Manufacturing Audit

### 8.1 Audit Records

| Record | Retention | Purpose |
| --- | --- | --- |
| Operator access logs | 5 years | Traceability |
| Provisioning records (per device) | Device lifetime + 5 years | Lifecycle tracking |
| Key ceremony records | Indefinite | Root of trust provenance |
| Test results (per batch) | 5 years | Quality assurance |
| Decommissioning records | 5 years | Device disposal verification |

### 8.2 Audit Review

- Monthly: Review operator access patterns for anomalies
- Per batch: Verify device count matches order quantity (no overproduction)
- Quarterly: Audit provisioning station integrity
- Annually: Full manufacturing security audit

---

## 9. Backend Integration

### 9.1 Pre-Provisioning Batch Authorization

Before a manufacturing batch, the backend must authorize:

- Batch size (max devices per batch)
- Allowed provisioning window (start/end time)
- Firmware version(s) permitted for provisioning
- Station IDs authorized for this batch

### 9.2 Post-Provisioning Verification

After provisioning, the backend verifies:

- Number of devices registered matches batch authorization
- No duplicate UIDs
- All lock attestations verified
- Provisioning completed within window

---

## 10. Requirements Trace

| Manufacturing Threat | SBOP Requirement | Control | Verification |
| --- | --- | --- | --- |
| MFR-T-001 (Overproduction) | REQ-SR-MFR-002 | Batch authorization, backend registration count | Batch audit |
| MFR-T-002 (Cloning) | REQ-SR-MFR-001, REQ-SR-ID-002 | UID from TRNG, attestation proof, backend dedup | Backend check |
| MFR-T-003 (Key extraction) | REQ-SR-MFR-001 | Encrypted key injection, KD wrapping | Crypto audit |
| MFR-T-004 (Unauthorized firmware) | REQ-SR-SC-001 | Signed test firmware, separate keys | Hash verification |
| MFR-T-005 (Non-provisioned) | REQ-SR-MFR-002 | Check lock state at end of provisioning line | Functional test |
| MFR-T-006 (Test persistence) | REQ-SR-DBG-001 | Lock verification; debug-disable fuse check | Fuse readback |
| MFR-T-007 (Data leak) | REQ-SR-MFR-002 | Audit log; encrypted storage | Audit review |
| MFR-T-008 (Counterfeit) | REQ-SR-ID-001 | UID uniqueness; backend registration check | Backend validation |
