# ECSS Compliance Mapping (Space Standards)

**Document ID:** CMP-ECSS-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document maps the SBOP platform to the European Cooperation for Space Standardization (ECSS) standards applicable to software and security for space systems.

Target standards:
- **ECSS-E-ST-40C**: Space Engineering -- Software
- **ECSS-Q-ST-80C**: Space Product Assurance -- Software Product Assurance
- **ECSS-E-ST-70-41C**: Space Engineering -- Telemetry and Telecommand Packet Utilization
- **ECSS-E-ST-50**: Communications (relevant security aspects)

---

## 2. Space Domain Context

### 2.1 Applicability

SBOP is applicable to space systems as a **critical onboard software component** responsible for:
- Boot-time firmware verification on spacecraft computers (OBC, payload computers)
- Secure telecommand (TC) authentication and integrity
- Flight software update management
- On-orbit reconfiguration security

### 2.2 Space-Specific Considerations

SBOP must address the following space-specific constraints:

| Constraint | Impact on SBOP | Mitigation |
| --- | --- | --- |
| Radiation environment | Single Event Upsets (SEU) may corrupt boot state | → Redundant state machine, TMR on critical paths |
| Extreme temperature | Component behavior variability | → Conservative timing margins |
| No physical access post-launch | Recovery must be fully remote | → Robust OTA recovery, dual-image with fallback |
| Long mission durations | Keys must outlast mission (5-15+ years) | → Key rotation, crypto agility |
| Limited bandwidth | OTA updates must be bandwidth-efficient | → Delta updates, compressed images |
| High latency | Authentication protocols must tolerate delay | → Non-interactive verification where possible |
| Power constraints | Boot must minimize energy use | → Early-exit on verification fail, no retry loops |

---

## 3. ECSS-E-ST-40C: Software Engineering

### 3.1 Clause 5.2: Software Requirements Engineering

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.2.2 | Software requirements specification | → `01_Requirements/` (complete REQ set) | Requirements_Overview.md |
| 5.2.3 | Requirements traceability | → `05_Verification/Traceability_Matrix.md` | Traceability |
| 5.2.4 | Requirements verification | → `01_Requirements/Requirements_Overview.md` (Section 8) | Verification methods |
| 5.2.5 | Software criticality classification | → This document (Section 8) | Criticality analysis |

### 3.2 Clause 5.3: Software Design

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.3.2 | Architectural design | → `00_Architecture/System_Overview.md` + `System_Decomposition.md` | Architecture definition |
| 5.3.3 | Detailed design | → `03_Subsystem/` (all subsystem specs) | Subsystem design |
| 5.3.4 | Interface definition | → `03_Subsystem/Boot/Boot_Interface.md` + other interfaces | Interface specs |
| 5.3.5 | Design justification | → `02_System_Design/System_Decomposition.md` (Section 2: Design Principles) | Design rationale |

### 3.3 Clause 5.4: Software Validation

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.4.2 | Validation plan | → `05_Verification/Test_Strategy.md` | Test strategy |
| 5.4.3 | Validation specification | → `05_Verification/Test_Cases.md` | Test cases |
| 5.4.4 | Validation execution | → Test execution results | To be produced |

### 3.4 Clause 5.5: Software Delivery and Acceptance

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.5.2 | Delivery documentation | → `07_Compliance/Work_Products.md` | Work products list |
| 5.5.3 | Acceptance testing | → `05_Verification/Test_Cases.md` | Acceptance criteria |

### 3.5 Clause 5.8: Software Maintenance

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.8.2 | Maintenance plan | → `06_Operations/OTA_Deployment.md` | OTA update strategy |
| 5.8.3 | Problem resolution | → `06_Operations/Incident_Response.md` | IR procedures |

---

## 4. ECSS-Q-ST-80C: Software Product Assurance

### 4.1 Clause 5.2: Software Product Assurance Programme

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.2.2 | SW PA programme | → `07_Compliance/Compliance_Overview.md` | Compliance strategy |
| 5.2.3 | SW PA plan | → `07_Compliance/Work_Products.md` | Work products tracking |
| 5.2.4 | SW PA implementation | → `07_Compliance/Review_and_Audit_Process.md` | Review process |

### 4.2 Clause 5.3: Software Process Assurance

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.3.2 | SW development process | → `08_Product/Reference_Architecture.md` | Reference implementation |
| 5.3.3 | SW verification process | → `05_Verification/Test_Strategy.md` | Verification strategy |
| 5.3.4 | SW validation process | → `05_Verification/Test_Cases.md` | Test cases |
| 5.3.5 | SW maintenance process | → `06_Operations/` | Operational procedures |

### 4.3 Clause 5.4: Software Product Assurance

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.4.2 | SW requirements quality | → `01_Requirements/Requirements_Overview.md` (Properties) | Requirements quality |
| 5.4.3 | SW design quality | → `02_System_Design/` | Design quality |
| 5.4.4 | SW code quality | Not applicable (Tier-1 spec) | — |
| 5.4.5 | SW documentation quality | → This document (review process) | Document review |

### 4.4 Clause 5.5: Software Quality Assurance

| ECSS Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.5.2 | SW quality metrics | → `05_Verification/Coverage_Model.md` | Coverage targets |
| 5.5.3 | SW non-conformance | → Gap analysis in compliance documents | Gap tracking |
| 5.5.4 | SW process improvement | → `07_Compliance/Review_and_Audit_Process.md` | Improvement cycle |

---

## 5. ECSS-E-ST-70-41C: Telemetry and Telecommand

### 5.1 TC Packet Authentication

SBOP's OTA update mechanism maps to ECSS telecommand (TC) security:

| ECSS Concept | SBOP Mapping | Description |
| --- | --- | --- |
| TC packet verification | → `03_Subsystem/Update/OTA_Interface.md` | Firmware update packet verification |
| TC authentication | → `03_Subsystem/Identity/Identity_Interface.md` | Device-to-backend authentication |
| TC sequence control | → `02_System_Design/State_Machine.md` (OTA states) | Ordered state transitions |
| TC execution verification | → `03_Subsystem/Update/OTA_State_Machine.md` | Update state verification |
| TC failure reporting | → `03_Subsystem/Update/OTA_Failure_Model.md` | OTA failure handling |

### 5.2 TM Reporting

| ECSS Concept | SBOP Mapping | Description |
| --- | --- | --- |
| TM packet generation | → `06_Operations/Monitoring_and_Telemetry.md` | Device status telemetry |
| TM housekeeping | → Boot success rate, version reporting | Operational parameters |
| TM event reporting | → Failure event telemetry | Anomaly reporting |

---

## 6. Criticality Classification

### 6.1 ECSS Criticality Categories

| Category | Definition | SBOP Component Classification |
| --- | --- | --- |
| **Category A** | Failure may cause loss of life or loss of spacecraft | Secure Boot |
| **Category B** | Failure may cause loss of mission | OTA Update Manager |
| **Category C** | Failure causes mission degradation | Identity verification |
| **Category D** | Failure causes minor inconvenience | Telemetry reporting |

### 6.2 SBOP Criticality Analysis

| Component | ECSS Category | Rationale |
| --- | --- | --- |
| Boot Subsystem | **Category A** | Failure to detect unauthorized firmware can lead to spacecraft loss |
| Crypto Subsystem | **Category A** | Crypto failure undermines all other security |
| Update Subsystem | **Category B** | Failed OTA may brick on-orbit asset |
| Identity Subsystem | **Category B** | Identity failure prevents authenticated communication |

### 6.3 Criticality-Based Requirements

| Category | Verification Rigor | SBOP Application |
| --- | --- | --- |
| A | Formal verification recommended | → `Formal_Verification.md` (Boot state machine) |
| A | 100% branch coverage | Boot verification logic |
| A | Independent V&V | → `Review_and_Audit_Process.md` |
| B | High coverage | OTA state machine testing |
| B | Fault injection recommended | → `Fault_Injection_Test.md` |

---

## 7. Space-Specific Security Hardening

### 7.1 Radiation Tolerance

| Requirement | Description | SBOP Impact |
| --- | --- | --- |
| RAD-001 | Critical state stored in TMR (Triple Modular Redundancy) | Boot state machine variables must be TMR-protected |
| RAD-002 | EDAC/ECC on all stored firmware images | Memory error detection requirement on storage abstraction |
| RAD-003 | Boot must verify integrity after SEU-triggered reset | Re-execute full verification on any unexpected reset |
| RAD-004 | No single point of failure in verification logic | Redundant comparison of verification results |

### 7.2 On-Orbit Recovery

| Requirement | Description | SBOP Impact |
| --- | --- | --- |
| REC-001 | Recovery without ground intervention | Dual-image with automatic fallback |
| REC-002 | Recovery after total power loss | Cold boot must reach OPERATIONAL from unpowered state |
| REC-003 | Minimal safe mode | Boot-only minimal mode for emergency recovery |

### 7.3 Crypto Longevity

| Requirement | Description | SBOP Impact |
| --- | --- | --- |
| CRY-001 | Key material must survive mission duration | Key derivation design must account for multi-year lifetimes |
| CRY-002 | Crypto agility for in-orbit algorithm updates | Signature algorithm identifier in image header enables migration |
| CRY-003 | Post-quantum migration path | Image format flags reserve bit for PQC signature type |

---

## 8. ECSS Clause-to-Evidence Trace

| ECSS Clause | SBOP Evidence | Gap | Action |
| --- | --- | --- | --- |
| E-ST-40 5.2.2 | `01_Requirements/` | — | — |
| E-ST-40 5.3.2 | `00_Architecture/` + `02_System_Design/` | — | — |
| E-ST-40 5.4.2 | `05_Verification/Test_Strategy.md` | No execution results | Execute test plans |
| E-ST-40 5.5.2 | `07_Compliance/Work_Products.md` | — | — |
| Q-ST-80 5.2.3 | `07_Compliance/Work_Products.md` | — | — |
| Q-ST-80 5.3.3 | `05_Verification/` | — | — |
| Q-ST-80 5.4.4 | Not applicable (Tier-1) | Implementation not in scope | — |
| E-ST-70-41 TC auth | `03_Subsystem/Identity/` | OTA authentication protocol not formalized | Formalize protocol |

---

## 9. Gap Analysis (Space Domain)

| Gap ID | Description | Impact | Priority |
| --- | --- | --- | --- |
| GAP-ECSS-001 | No TMR/EDAC specification for radiation-hardened deployment | Critical for deep-space missions | P1 |
| GAP-ECSS-002 | No formal TC authentication protocol specification | Required for ECSS-E-70-41 compliance | P1 |
| GAP-ECSS-003 | No independent V&V plan for Category A components | Required for ECSS criticality A | P1 |
| GAP-ECSS-004 | No crypto longevity analysis (multi-year mission duration) | Operational risk over long missions | P2 |
| GAP-ECSS-005 | No post-quantum crypto migration path documented | Future compliance; not yet required | P3 |
| GAP-ECSS-006 | No on-orbit recovery without ground intervention plan | Required for deep-space autonomy | P2 |

---

## 10. References

- ECSS-E-ST-40C: Space Engineering - Software
- ECSS-Q-ST-80C: Space Product Assurance - Software Product Assurance
- ECSS-E-ST-70-41C: Telemetry and Telecommand Packet Utilization
- ECSS-E-ST-50-03C: Space Data Links - Telecommand Protocols
