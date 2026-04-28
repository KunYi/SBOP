# Security Controls

**Document ID:** SEC-CTRL-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Preventive Controls

| Control ID | Control Name | Description | Domain | Verification |
| --- | --- | --- | --- | --- |
| CTRL-PRV-001 | Firmware Signature Verification | Cryptographically verify firmware authenticity before execution | Boot | TEST-BOOT-002 |
| CTRL-PRV-002 | Firmware Integrity Verification | SHA-256 hash verification of firmware payload | Boot | TEST-BOOT-003 |
| CTRL-PRV-003 | Key Isolation | Keys stored in secure element or protected memory; never exposed | Crypto | ANALYSIS |
| CTRL-PRV-004 | Version Enforcement | Monotonic version counter; reject rollback | Boot | TEST-BOOT-004 |
| CTRL-PRV-005 | Debug Port Disable | JTAG/SWD disabled after provisioning; authenticated unlock only | Debug | Functional test |
| CTRL-PRV-006 | Commit Signing | All source commits must be cryptographically signed | Supply Chain | Audit |
| CTRL-PRV-007 | Reproducible Builds | Build output must be bit-identical given same inputs | Supply Chain | Independent build |
| CTRL-PRV-008 | SBOM Generation | Software Bill of Materials generated per build | Supply Chain | SBOM audit |
| CTRL-PRV-009 | Constant-Time Crypto | All crypto and comparison ops execute in constant time | Side-Channel | Timing measurement |
| CTRL-PRV-010 | Glitch Detection | Voltage/clock monitors detect fault injection attempts | Physical | Fault injection test |
| CTRL-PRV-011 | Secure Provisioning | Key injection over encrypted channel in controlled facility | Manufacturing | Factory audit |
| CTRL-PRV-012 | Device Attestation | Device proves identity via KD-based challenge-response | Identity | TEST-ID-001 |
| CTRL-PRV-013 | Encrypted Storage | Firmware stored encrypted at rest (optional, per flags) | Crypto | TEST-CRYPTO-003 |
| CTRL-PRV-014 | Memory Protection | Zone 1 memory not accessible from Zone 2 | Boot | ANALYSIS |

---

## 2. Detective Controls

| Control ID | Control Name | Description | Domain | Verification |
| --- | --- | --- | --- | --- |
| CTRL-DET-001 | Backend Anomaly Detection | Monitor update request patterns for anomalies | Backend | Operational monitoring |
| CTRL-DET-002 | Telemetry Monitoring | Device health status, boot/OTA success rate | Operations | Operational monitoring |
| CTRL-DET-003 | Tamper Detection | Enclosure switches, environmental sensors | Physical | Physical pentest |
| CTRL-DET-004 | Debug Event Logging | All debug access attempts logged | Debug | Audit review |
| CTRL-DET-005 | Provisioning Audit | All provisioning operations logged with operator attribution | Manufacturing | Audit review |
| CTRL-DET-006 | CVE Scanning | Dependencies scanned for known vulnerabilities per build | Supply Chain | CVE scan report |
| CTRL-DET-007 | Firmware Integrity Monitoring | Periodic integrity check of stored firmware images | Boot | TEST-BOOT-005 |
| CTRL-DET-008 | Identity Anomaly Detection | Duplicate UID detection at backend | Identity | Backend validation |

---

## 3. Corrective Controls

| Control ID | Control Name | Description | Domain | Verification |
| --- | --- | --- | --- | --- |
| CTRL-COR-001 | OTA Rollback | Revert to previous valid firmware if new version fails | Update | TEST-OTA-004 |
| CTRL-COR-002 | Key Revocation | Revoke and replace compromised keys | Crypto / Operations | Key lifecycle audit |
| CTRL-COR-003 | Forced Update | Backend forces update of vulnerable firmware | Update | Operational test |
| CTRL-COR-004 | Device Lockdown | Remote lockdown of compromised device | Identity | Operational test |
| CTRL-COR-005 | Key Zeroization | Immediate key destruction on critical tamper detect | Physical | Physical pentest |
| CTRL-COR-006 | Manufacturing Rework | Re-provision devices with provisioning failure | Manufacturing | Factory audit |
| CTRL-COR-007 | Incident Response | Structured response to security incidents (detect, contain, recover) | Operations | → `Incident_Response.md` |

---

## 4. Control Coverage by Domain

| Domain | Preventive | Detective | Corrective |
| --- | --- | --- | --- |
| Boot Security | CTRL-PRV-001, 002, 004, 014 | CTRL-DET-007 | CTRL-COR-001 |
| Update Security | CTRL-PRV-012 | CTRL-DET-001, 002 | CTRL-COR-001, 003, 004 |
| Identity Security | CTRL-PRV-012 | CTRL-DET-008 | CTRL-COR-004 |
| Crypto Security | CTRL-PRV-003, 009, 013 | — | CTRL-COR-002, 005 |
| Supply Chain | CTRL-PRV-006, 007, 008 | CTRL-DET-006 | — |
| Physical Security | CTRL-PRV-010 | CTRL-DET-003 | CTRL-COR-005 |
| Debug Security | CTRL-PRV-005 | CTRL-DET-004 | — |
| Manufacturing | CTRL-PRV-011 | CTRL-DET-005 | CTRL-COR-006 |
| Operations (all) | — | CTRL-DET-002 | CTRL-COR-007 |

---

## 5. Control-to-Requirement Trace

→ See `Security_Trace.md` and `Enhanced_Security_Trace.md` for full traceability from threats through controls to requirements and tests.
