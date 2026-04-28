# Medical Device Software Compliance Mapping

**Document ID:** CMP-MED-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document maps the SBOP platform to medical device software standards:

- **IEC 62304:2006 + A1:2015**: Medical Device Software -- Software Life Cycle Processes
- **ISO 14971:2019**: Medical Devices -- Application of Risk Management to Medical Devices
- **ISO 13485:2016**: Medical Devices -- Quality Management Systems (relevant aspects)

It also addresses considerations for **FDA 21 CFR Part 820** (Quality System Regulation) and **EU MDR 2017/745** (Medical Device Regulation) software requirements.

---

## 2. IEC 62304 Software Safety Classification

### 2.1 Classification Criteria

| Class | Definition | SBOP Applicability |
| --- | --- | --- |
| **Class A** | No injury or damage to health | Non-medical deployments of SBOP |
| **Class B** | Non-serious injury possible | SBOP for non-critical medical devices (e.g., diagnostic tools) |
| **Class C** | Death or serious injury possible | SBOP for life-sustaining devices (e.g., infusion pumps, ventilators, implantable devices) |

### 2.2 SBOP Classification by Deployment

| Deployment Scenario | IEC 62304 Class | Example |
| --- | --- | --- |
| Standalone diagnostic | Class B | Firmware in a blood glucose meter |
| Therapeutic device | Class B/C | Firmware in an insulin pump |
| Life-sustaining device | Class C | Firmware in a ventilator or pacemaker |
| Implantable device | Class C | Firmware in an implantable cardioverter-defibrillator (ICD) |

SBOP targets **Class C** capability, as the same platform may be deployed across all medical device classes.

---

## 3. IEC 62304 Clause Mapping

### 3.1 Clause 4: General Requirements

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 4.2 | Quality management system | → `07_Compliance/Compliance_Overview.md` | Compliance framework |
| 4.3 | Risk management | → `04_Security/Risk_Quantification.md` + `Risk_Treatment.md` | Risk framework |
| 4.4 | Software safety classification | → This document (Section 2) | Safety classification |

### 3.2 Clause 5: Software Development Process

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.1.1 | Software development plan | → `07_Compliance/Work_Products.md` | Work products |
| 5.1.2 | Software development plan content | → `08_Product/Reference_Architecture.md` | Development plan |
| 5.1.3 | Development plan updates | → `07_Compliance/Review_and_Audit_Process.md` | Review cycle |
| 5.2.1 | Software requirements analysis | → `01_Requirements/` | Requirements |
| 5.2.2 | Software requirements content | → `01_Requirements/Requirements_Overview.md` | Requirements properties |
| 5.2.3 | Risk control measures in requirements | → `01_Requirements/REQ-SR-SECURITY.md` | Security requirements |
| 5.2.4 | Requirements review | → `07_Compliance/Review_and_Audit_Process.md` | Review process |

### 3.3 Clause 5.3: Software Architectural Design

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.3.1 | Architectural design plan | → `00_Architecture/System_Overview.md` | System overview |
| 5.3.2 | Architectural design content | → `02_System_Design/System_Decomposition.md` | Decomposition |
| 5.3.3 | Architecture for risk control | → `04_Security/Security_Controls.md` | Security controls |
| 5.3.4 | Interface specification | → `03_Subsystem/` (all interfaces) | Interface docs |
| 5.3.5 | Segregation (Class C) | → `02_System_Design/System_Decomposition.md` (Section 5: Failure Isolation) | Component isolation |
| 5.3.6 | Architectural design review | → `Review_and_Audit_Process.md` | Review process |

### 3.4 Clause 5.4: Software Detailed Design

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.4.2 | Detailed design content | → `03_Subsystem/` (subsystem detail docs) | Subsystem design |
| 5.4.3 | Detailed design for risk control | → `04_Security/Mitigation_Strategy.md` | Mitigation design |
| 5.4.4 | Detailed interfaces | → `03_Subsystem/Boot/Boot_Interface.md` | Boot interface |

### 3.5 Clause 5.5: Software Unit Implementation and Verification

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.5.1 | Unit implementation plan | → Not applicable (Tier-1) | — |
| 5.5.2 | Unit acceptance criteria | → `05_Verification/Test_Cases.md` | Test acceptance |
| 5.5.3 | Unit verification | → `05_Verification/Test_Strategy.md` (Unit Level) | Unit testing |
| 5.5.4 | Additional Class C unit verification | → `05_Verification/Coverage_Model.md` (100% target) | Coverage requirements |

### 3.6 Clause 5.6: Software Integration and Integration Testing

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.6.2 | Integration plan | → `05_Verification/Test_Strategy.md` (Subsystem + System Level) | Integration strategy |
| 5.6.3 | Integration testing | → `05_Verification/Test_Cases.md` | Integration tests |

### 3.7 Clause 5.7: Software System Testing

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.7.1 | System test plan | → `05_Verification/Test_Strategy.md` (System Level) | System test strategy |
| 5.7.2 | System test procedures | → `05_Verification/Test_Cases.md` | Test procedures |
| 5.7.3 | System test pass/fail criteria | → `05_Verification/Test_Strategy.md` (Acceptance Criteria) | Acceptance criteria |

### 3.8 Clause 5.8: Software Release

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.8.1 | Software release process | → `06_Operations/OTA_Deployment.md` | Deployment process |
| 5.8.2 | Release documentation | → `07_Compliance/Work_Products.md` | Work products |
| 5.8.3 | Archiving | → `07_Compliance/Evidence_List.md` (Storage) | Evidence archival |

### 3.9 Clause 6: Software Maintenance Process

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 6.1 | Maintenance plan | → `06_Operations/OTA_Deployment.md` | OTA strategy |
| 6.2 | Problem and modification analysis | → `06_Operations/Incident_Response.md` | IR procedures |
| 6.3 | Modification implementation | → `06_Operations/OTA_Deployment.md` | OTA deployment |

### 3.10 Clause 7: Software Configuration Management

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 7.2 | Configuration identification | → Document ID scheme across all docs | Configuration IDs |
| 7.3 | Change control | → Deterministic build requirement (REQ-NFR-SYS-003) | Change management |
| 7.4 | Configuration status accounting | → `07_Compliance/Work_Products.md` | Version tracking |

### 3.11 Clause 8: Software Problem Resolution

| IEC 62304 Clause | Requirement | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 8.2 | Problem reporting | → `06_Operations/Monitoring_and_Telemetry.md` | Anomaly detection |
| 8.3 | Problem investigation | → `06_Operations/Incident_Response.md` | IR investigation |
| 8.4 | Problem resolution | → `06_Operations/OTA_Deployment.md` | Update deployment |
| 8.5 | Trend analysis | → `06_Operations/Monitoring_and_Telemetry.md` (Metrics) | Trend monitoring |

---

## 4. ISO 14971 Risk Management

### 4.1 Risk Management Process

ISO 14971 defines a risk management process for medical devices. SBOP maps as follows:

| ISO 14971 Clause | Activity | SBOP Mapping |
| --- | --- | --- |
| 4.2 | Risk management plan | → `04_Security/Risk_Quantification.md` |
| 4.3 | Risk management file | → `04_Security/` (entire security section) |
| 5.1 | Risk analysis | → `04_Security/Threat_Model.md` + `Attack_Tree.md` |
| 5.2 | Intended use and safety characteristics | → `00_Architecture/System_Overview.md` (Objectives) |
| 5.3 | Identification of hazards | → `04_Security/Attack_Surface.md` (entry points) |
| 5.4 | Estimation of risks | → `04_Security/Risk_Quantification.md` |
| 6 | Risk evaluation | → `04_Security/Risk_Quantification.md` (Risk Table) |
| 7.1 | Risk control option analysis | → `04_Security/Risk_Treatment.md` |
| 7.2 | Implementation of risk control measures | → `04_Security/Mitigation_Strategy.md` |
| 7.3 | Residual risk evaluation | → `04_Security/Residual_Risk.md` |
| 7.4 | Benefit-risk analysis | → `04_Security/Risk_Treatment.md` (Accept justification) |
| 7.5 | Risks from control measures | → `04_Security/Security_Controls.md` |
| 7.6 | Completeness of risk control | → `04_Security/Security_Trace.md` (Coverage Rules) |
| 8 | Evaluation of overall residual risk | → `04_Security/Residual_Risk.md` |
| 9 | Risk management review | → `07_Compliance/Review_and_Audit_Process.md` |
| 10 | Production and post-production activities | → `06_Operations/` (all operational docs) |

### 4.2 Hazard Severity Levels

Aligned with ISO 14971:

| Severity | Description | SBOP Impact Example |
| --- | --- | --- |
| Catastrophic (4) | Death or permanent impairment | Unauthorized firmware in life-sustaining device |
| Critical (3) | Serious injury or reversible impairment | Tampered firmware causing overdose |
| Serious (2) | Minor injury requiring intervention | Device malfunction requiring replacement |
| Minor (1) | No injury; temporary inconvenience | Telemetry data loss |

### 4.3 Hazard Identification for SBOP

| Hazard ID | Hazard Description | Hazardous Situation | Severity |
| --- | --- | --- | --- |
| HZ-001 | Unauthorized firmware execution | Device performs unintended therapy | Catastrophic (4) |
| HZ-002 | Firmware corruption during update | Device bricked; therapy interrupted | Critical (3) |
| HZ-003 | Rollback to vulnerable firmware | Previously fixed vulnerability re-exploited | Catastrophic (4) |
| HZ-004 | Key extraction | Attacker signs malicious firmware | Catastrophic (4) |
| HZ-005 | Device identity cloning | Counterfeit device in clinical use | Catastrophic (4) |
| HZ-006 | OTA interruption | Device in undefined state | Serious (2) |

### 4.4 Risk Control Mapping

| Hazard | Risk Control | SBOP Requirement | Verification |
| --- | --- | --- | --- |
| HZ-001 | Signature verification | REQ-SR-BOOT-001 | TEST-BOOT-002 |
| HZ-002 | Dual-image + rollback | REQ-FR-OTA-004, REQ-FR-OTA-005 | TEST-OTA-002, TEST-OTA-004 |
| HZ-003 | Version enforcement | REQ-FR-BOOT-004, REQ-SR-OTA-002 | TEST-BOOT-004 |
| HZ-004 | Key isolation | REQ-SR-KEY-001 | ANALYSIS |
| HZ-005 | Identity binding | REQ-SR-ID-001, REQ-SR-ID-002 | TEST-ID-002 |
| HZ-006 | Recovery mechanism | REQ-FR-OTA-004 | TEST-OTA-002 |

---

## 5. SOUP (Software of Unknown Provenance)

### 5.1 SOUP Identification

Per IEC 62304, SBOP must identify and assess third-party software integrated into the platform:

| Component Type | Examples | SOUP Assessment |
| --- | --- | --- |
| Cryptographic library | ECDSA/SHA-256 implementation | Must be validated (FIPS 140-3 or equivalent) |
| TLS stack | mbedTLS, wolfSSL | Must be validated for medical use |
| RTOS | FreeRTOS, Zephyr (if used) | Safety certification status |
| Hardware abstraction | MCU vendor HAL | Assess for safety impact |

### 5.2 SOUP Risk Assessment Approach

For each SOUP component:
1. Identify anomaly lists (known defects from vendor)
2. Assess impact on SBOP safety classification
3. Define controls to mitigate SOUP anomalies
4. Monitor for new anomalies during maintenance

---

## 6. FDA / EU MDR Considerations

### 6.1 FDA 21 CFR Part 820.30 (Design Controls)

| FDA Requirement | SBOP Mapping |
| --- | --- |
| Design and development planning | `08_Product/Reference_Architecture.md` |
| Design input | `01_Requirements/` |
| Design output | `02_System_Design/` + `03_Subsystem/` |
| Design review | `07_Compliance/Review_and_Audit_Process.md` |
| Design verification | `05_Verification/Test_Cases.md` |
| Design validation | `05_Verification/Red_Team_Test_Plan.md` |
| Design transfer | `06_Operations/Provisioning_Operations.md` |
| Design changes | `06_Operations/OTA_Deployment.md` |
| Design history file | `07_Compliance/Evidence_List.md` |

### 6.2 EU MDR 2017/745 Annex I (General Safety and Performance Requirements)

| EU MDR GSPR | SBOP Mapping |
| --- | --- |
| 17.1: SW verification and validation | `05_Verification/` |
| 17.2: SW development lifecycle | `07_Compliance/Compliance_Overview.md` |
| 17.3: SW security (IT security) | `04_Security/` + `00_Architecture/Security_Overview.md` |
| 17.4: SW identification | SBOP versioning in image header |

---

## 7. Medical Device Evidence Package

### 7.1 Required Artifacts for IEC 62304 Submission

| Artifact | SBOP Document | Class B | Class C |
| --- | --- | --- | --- |
| Software development plan | `Work_Products.md` | Required | Required |
| Software requirements specification | `01_Requirements/` | Required | Required |
| Software architecture | `00_Architecture/` + `02_System_Design/` | Required | Required |
| Software detailed design | `03_Subsystem/` | Not required | **Required** |
| Software unit test results | `Test_Cases.md` (Unit) | Required | **Required (100%)** |
| Software integration test results | `Test_Cases.md` (Integration) | Not required | **Required** |
| Software system test results | `Test_Cases.md` (System) | Required | Required |
| Risk management file | `04_Security/` (complete) | Required | Required |
| SOUP assessment | This document (Section 5) | Required | Required |
| Problem resolution records | `Incident_Response.md` | Required | Required |
| Maintenance plan | `OTA_Deployment.md` | Required | Required |

### 7.2 Audit Checklist

- [ ] Software safety classification documented and justified
- [ ] Risk management file is complete per ISO 14971
- [ ] ALL requirements are traceable to design, implementation, and test
- [ ] SOUP components identified and assessed
- [ ] Problem resolution process defined
- [ ] Maintenance (update) process defined
- [ ] Configuration management in place
- [ ] For Class C: detailed design documented; 100% unit coverage

---

## 8. Gap Analysis (Medical Domain)

| Gap ID | Description | Impact | Priority |
| --- | --- | --- | --- |
| GAP-MED-001 | No formal SOUP anomaly list for crypto/TLS dependencies | Required for IEC 62304 SOUP assessment | P1 |
| GAP-MED-002 | No design history file (DHF) per FDA 820.30 | Required for FDA submission | P1 |
| GAP-MED-003 | No hazard traceability matrix (hazard → risk control → verification) | Required for ISO 14971 | P1 |
| GAP-MED-004 | No software of unknown provenance validation results | Required for IEC 62304 | P2 |
| GAP-MED-005 | No segregation verification for Class C (Zone 1 / Zone 2 boundary) | Required for IEC 62304 5.3.5 | P1 |
| GAP-MED-006 | No clinical evaluation of cybersecurity impact | Required for EU MDR | P3 |
