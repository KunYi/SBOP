# IEC 62443 Detailed Compliance Mapping

**Document ID:** CMP-IEC62443-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document provides the detailed clause-by-clause mapping between the SBOP platform and IEC 62443 *Industrial communication networks -- Network and system security*.

It addresses:
- **IEC 62443-4-1**: Secure product development lifecycle
- **IEC 62443-4-2**: Technical security requirements for IACS components
- **IEC 62443-3-3**: System security requirements and security levels

---

## 2. Foundation Concepts (IEC 62443-1-1)

### 2.1 Zones and Conduits

SBOP defines security zones and communication conduits per IEC 62443-1-1.

→ See `04_Security/Zone_Conduit_Model.md`

### 2.2 Security Levels

IEC 62443 defines four security levels:

| Level | Name | Description |
| --- | --- | --- |
| SL 1 | Basic | Protection against casual or coincidental violation |
| SL 2 | Enhanced | Protection against intentional violation using simple means with low resources, generic skills, and low motivation |
| SL 3 | High | Protection against intentional violation using sophisticated means with moderate resources, IACS-specific skills, and moderate motivation |
| SL 4 | Critical | Protection against intentional violation using sophisticated means with extended resources, IACS-specific skills, and high motivation |

SBOP targets **SL 2** with capability for **SL 3** when deployed with a hardware secure element.

→ See `04_Security/CAL_SLT_Definitions.md` for per-zone SL-T assignments.

---

## 3. Foundational Requirements (FR) Mappings

### 3.1 FR 1: Identification and Authentication Control (IAC)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 1.1 | Human user identification and authentication | → `06_Operations/Key_Management.md` (operator auth) | SL 2 |
| CR 1.2 | Software/device identification and authentication | → `03_Subsystem/Identity/Identity_Overview.md` | SL 3 |
| CR 1.3 | Account management | → `06_Operations/Provisioning_Operations.md` | SL 2 |
| CR 1.4 | Identifier management | → `03_Subsystem/Identity/Identity_Interface.md` | SL 3 |
| CR 1.5 | Authenticator management | → `06_Operations/Key_Management.md` (key lifecycle) | SL 2 |
| CR 1.6 | Wireless access management | N/A (SBOP does not manage wireless) | — |
| CR 1.7 | Strength of password-based authentication | → `03_Subsystem/Crypto/Crypto_Algorithms.md` (key strength) | SL 3 |
| CR 1.8 | Public key infrastructure certificates | → `02_System_Design/Key_Hierarchy.md` | SL 2 |
| CR 1.9 | Strength of public key authentication | → ECDSA P-256 / Ed25519 per `Crypto_Algorithms.md` | SL 3 |
| CR 1.10 | Authenticator feedback | N/A (no interactive auth on device) | — |
| CR 1.11 | Unsuccessful login attempts | → `03_Subsystem/Identity/Identity_Failure_Model.md` | SL 2 |
| CR 1.12 | System use notification | N/A | — |
| CR 1.13 | Access via untrusted networks | → `02_System_Design/Data_Flow.md` (trust boundaries) | SL 2 |

### 3.2 FR 2: Use Control (UC)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 2.1 | Authorization enforcement | → `00_Architecture/Trust_Model.md` (trust rules) | SL 3 |
| CR 2.2 | Wireless use control | N/A | — |
| CR 2.3 | Use control for portable and mobile devices | N/A | — |
| CR 2.4 | Mobile code | N/A | — |
| CR 2.5 | Session lock | N/A (no interactive sessions) | — |
| CR 2.6 | Remote session termination | N/A | — |
| CR 2.7 | Concurrent session control | N/A | — |
| CR 2.8 | Auditable events | → `06_Operations/Monitoring_and_Telemetry.md` | SL 2 |
| CR 2.9 | Audit storage capacity | → `06_Operations/Monitoring_and_Telemetry.md` (data collection) | SL 1 |
| CR 2.10 | Response to audit processing failures | → `06_Operations/Incident_Response.md` | SL 2 |
| CR 2.11 | Timestamps | Required for all log entries | SL 2 |
| CR 2.12 | Non-repudiation | → `02_System_Design/Image_Format.md` (signature non-repudiation) | SL 2 |

### 3.3 FR 3: System Integrity (SI)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 3.1 | Communication integrity | → `02_System_Design/Data_Flow.md` | SL 2 |
| CR 3.2 | Malicious code protection | → `01_Requirements/REQ-FR-BOOT-001` (no unauthorized code) | SL 3 |
| CR 3.3 | Security functionality verification | → `03_Subsystem/Boot/Boot_Overview.md` | SL 3 |
| CR 3.4 | SW and information integrity | → `01_Requirements/REQ-FR-BOOT-002` (integrity check) | SL 3 |
| CR 3.5 | Input validation | → `02_System_Design/Image_Format.md` (Section 11, validation rules) | SL 3 |
| CR 3.6 | Deterministic output | → `03_Subsystem/Boot/Boot_Failure_Model.md` (fail-safe) | SL 2 |
| CR 3.7 | Error handling | → `02_System_Design/Error_Code_Catalog.md` (Phase 4.1) | SL 2 |
| CR 3.8 | Session integrity | N/A | — |
| CR 3.9 | Protection of audit information | → `07_Compliance/Evidence_List.md` (tamper-evident) | SL 2 |

### 3.4 FR 4: Data Confidentiality (DC)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 4.1 | Information confidentiality | → `03_Subsystem/Crypto/Crypto_Algorithms.md` (optional encryption) | SL 2 |
| CR 4.2 | Information persistence | N/A (no persistent data at boot level) | — |
| CR 4.3 | Use of cryptography | → `03_Subsystem/Crypto/Crypto_Overview.md` | SL 3 |

### 3.5 FR 5: Restricted Data Flow (RDF)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 5.1 | Network segmentation | → `04_Security/Zone_Conduit_Model.md` | SL 2 |
| CR 5.2 | Zone boundary protection | → `00_Architecture/Trust_Model.md` (trust boundaries) | SL 2 |
| CR 5.3 | General purpose communication restrictions | → `02_System_Design/System_Decomposition.md` (explicit interfaces) | SL 2 |

### 3.6 FR 6: Timely Response to Events (TRE)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 6.1 | Audit log accessibility | → `07_Compliance/Audit_Trace.md` | SL 2 |
| CR 6.2 | Continuous monitoring | → `06_Operations/Monitoring_and_Telemetry.md` | SL 2 |

### 3.7 FR 7: Resource Availability (RA)

| 62443-4-2 CR | Description | SBOP Mapping | SL Achieved |
| --- | --- | --- | --- |
| CR 7.1 | DoS protection | → `03_Subsystem/Boot/Boot_Failure_Model.md` (fail-safe) | SL 1 |
| CR 7.2 | Resource management | → `01_Requirements/REQ-NFR-SYS-005` (constrained resources) | SL 2 |
| CR 7.3 | Control system backup | → `02_System_Design/System_Flow.md` (dual-image) | SL 2 |
| CR 7.4 | Control system recovery and reconstitution | → `03_Subsystem/Update/OTA_Recovery.md` | SL 2 |
| CR 7.5 | Emergency power | N/A (platform-dependent) | — |
| CR 7.6 | Network and security configuration settings | N/A | — |
| CR 7.7 | Least functionality | → `00_Architecture/Security_Overview.md` (least privilege) | SL 2 |
| CR 7.8 | Control system component inventory | → `06_Operations/Provisioning_Operations.md` (device record) | SL 2 |

---

## 4. IEC 62443-4-1: Secure Product Development Lifecycle

| 62443-4-1 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| SM-1 | Security management | → `07_Compliance/Compliance_Overview.md` | Compliance framework |
| SM-2 | Security requirements | → `01_Requirements/` (complete REQ set) | Requirements trace |
| SM-3 | Security by design | → `02_System_Design/` (all docs) | Design artifacts |
| SM-4 | Secure implementation | → `03_Subsystem/` | Subsystem specs |
| SM-5 | Security verification and validation | → `05_Verification/Test_Strategy.md` | Verification plan |
| SM-6 | Security defect management | → `06_Operations/Incident_Response.md` | IR procedures |
| SM-7 | Security update management | → `06_Operations/OTA_Deployment.md` | Update deployment |
| SM-8 | Security guidelines | → `00_Architecture/Security_Overview.md` | Security principles |
| SM-9 | Supplier security | → `04_Security/Manufacturing_Security.md` (Phase 3.5) | Supplier requirements |
| SM-10 | Security review | → `07_Compliance/Review_and_Audit_Process.md` | Audit process |
| SM-11 | Security testing | → `05_Verification/` | Test plans/cases |
| SM-12 | Penetration testing | → `05_Verification/Red_Team_Test_Plan.md` | Red team testing |
| SM-13 | Component security | → `03_Subsystem/` (per-subsystem specs) | Component design |

---

## 5. SL Achievement by Zone

| Zone | SL-T | SL-A (Current) | Gap |
| --- | --- | --- | --- |
| Device Core (Zone 1) | SL 3 | SL 2 | Need hardware secure element for SL 3 key protection |
| Device Application (Zone 2) | SL 2 | SL 2 | — |
| Backend Infrastructure (Zone 3) | SL 3 | SL 2 | Need formal backend security assessment |
| Manufacturing (Zone 4) | SL 2 | SL 1 | → `Manufacturing_Security.md` (Phase 3.5) |
| Network (Zone 5) | SL 2 | SL 1 | Need transport security specification |
| Toolchain (Zone 6) | SL 3 | SL 1 | → `Supply_Chain_Security.md` (Phase 3.1) |

---

## 6. System Requirement (SR) Mappings (IEC 62443-3-3)

| SR | Description | SBOP Mapping |
| --- | --- | --- |
| SR 1.1 | Human user authentication | N/A (no human users on device) |
| SR 1.2 | SW process/device authentication | `Identity_Overview.md` + `Identity_Interface.md` |
| SR 1.3 | Account management | `Key_Management.md` |
| SR 2.1 | Authorization enforcement | `Trust_Model.md` |
| SR 3.1 | Communication integrity | `Data_Flow.md` + crypto subsystem |
| SR 3.2 | Malicious code protection | `Boot_Overview.md` + signature verification |
| SR 4.1 | Information confidentiality | Crypto subsystem (encryption flag in image header) |
| SR 5.1 | Network segmentation | `Zone_Conduit_Model.md` |
| SR 6.1 | Audit log accessibility | `Audit_Trace.md` |
| SR 7.1 | DoS protection | `Boot_Failure_Model.md` + `OTA_Recovery.md` |

---

## 7. Consolidated Evidence Matrix

| IEC 62443 Document | SBOP Evidence | Status |
| --- | --- | --- |
| 62443-4-1 (SDLC) | All SBOP artifacts (WP-01 through WP-13) | Partial |
| 62443-4-2 (Component) | This document (FR mappings) | Defined |
| 62443-3-3 (System) | This document (SR mappings) | Defined |
| 62443-2-1 (Security program) | `Compliance_Overview.md` | Partial |
| 62443-2-4 (Service provider) | `Manufacturing_Security.md` | Planned |

---

## 8. Gap Analysis

| Gap ID | Description | Impact | Mitigation |
| --- | --- | --- | --- |
| GAP-62443-001 | No formal SL-C (capability) rating per component | Medium | → `CAL_SLT_Definitions.md` (Phase 2.5) |
| GAP-62443-002 | Toolchain security (Zone 6) currently SL 1; target SL 3 | High | → `Supply_Chain_Security.md` (Phase 3.1) |
| GAP-62443-003 | Manufacturing zone (Zone 4) SL-A below SL-T | Medium | → `Manufacturing_Security.md` (Phase 3.5) |
| GAP-62443-004 | No transport security specification for Network zone | Medium | → Network conduit security requirements |
| GAP-62443-005 | No DoS protection beyond fail-safe | Low | Resource limits specification |
