# Evidence List

**Document ID:** CMP-EVI-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the complete catalog of evidence required to support SBOP compliance claims across all 6 standards. Evidence is organized by category and mapped to requirements, standards clauses, and verification activities.

---

## 2. Evidence Categories

### 2.1 Design Evidence (EVI-DES)

| ID | Evidence Item | Source Document | Requirements Addressed | Standards |
| --- | --- | --- | --- | --- |
| EVI-DES-001 | System architecture specification | `System_Overview.md`, `Reference_Architecture.md` | All REQ-FR, REQ-SR | All |
| EVI-DES-002 | Security goals and traceability | `Security_Goals.md` | G-AUTH through G-MFR | ISO 21434 §9, IEC 62443 CR 1.1 |
| EVI-DES-003 | Attack tree (all 53 leaf nodes) | `Attack_Tree.md` | — | ISO 21434 §9.5, IEC 62443 CR 3.1 |
| EVI-DES-004 | TARA risk assessment | `TARA_Methodology.md`, `Risk_Quantification.md` | — | ISO 21434 §9.5 |
| EVI-DES-005 | Zone and conduit model | `Zone_Conduit_Model.md` | REQ-SR-BOOT-001, REQ-SR-KEY-001 | IEC 62443 §4.2 |
| EVI-DES-006 | Key hierarchy and management | `Key_Hierarchy.md`, `Key_Management.md` | REQ-SR-KEY-001, REQ-SR-KEY-002 | IEC 62443 CR 1.3, ISO 21434 §9.7 |
| EVI-DES-007 | Interface contracts | `Interface_Contracts.md` | All REQ-FR, REQ-SR | All |
| EVI-DES-008 | Error code catalog | `Error_Code_Catalog.md` | All | IEC 62304 §5.3 |
| EVI-DES-009 | Data structure definitions | `Data_Structures.md` | REQ-FR-BOOT-001, REQ-FR-OTA-001 | All |
| EVI-DES-010 | Architecture decision records | `Architecture_Decision_Record.md` | All | All |
| EVI-DES-011 | Safety analysis | `Safety_Analysis.md` | — | ISO 26262, IEC 61508, DO-178C |
| EVI-DES-012 | Formal verification specification | `Formal_Verification.md` | REQ-SR-BOOT-001 | DO-178C DO-333, IEC 61508-7 |

### 2.2 Implementation Evidence (EVI-IMP)

| ID | Evidence Item | Source | Requirements Addressed | Standards |
| --- | --- | --- | --- | --- |
| EVI-IMP-001 | Bootloader source code | Reference implementation | REQ-FR-BOOT-001..005 | All |
| EVI-IMP-002 | Crypto engine source code | Reference implementation | REQ-SR-BOOT-001, REQ-SR-SCH-001 | All |
| EVI-IMP-003 | OTA client source code | Reference implementation | REQ-FR-OTA-001..005 | All |
| EVI-IMP-004 | Build pipeline configuration | CI/CD config | REQ-SR-SC-001 | ISO 21434 §10 |
| EVI-IMP-005 | SBOM (SPDX + CycloneDX) | Build output | REQ-SR-SC-001 | ISO 21434 §9.8, IEC 62304 §5.3 |
| EVI-IMP-006 | Reproducible build verification | Build pipeline logs | REQ-SR-SC-001 | ISO 21434 §10 |
| EVI-IMP-007 | Static analysis report | CI output | All | IEC 61508-3 §7.4, DO-178C |
| EVI-IMP-008 | MISRA compliance report | Static analysis | All | ISO 26262-6, IEC 61508-3 |

### 2.3 Verification Evidence (EVI-VER)

| ID | Evidence Item | Source | Requirements Addressed | Standards |
| --- | --- | --- | --- | --- |
| EVI-VER-001 | Boot test results (TEST-BOOT-001..005) | Test execution | REQ-FR-BOOT-001..005 | All |
| EVI-VER-002 | OTA test results (TEST-OTA-001..005) | Test execution | REQ-FR-OTA-001..005 | All |
| EVI-VER-003 | Identity test results (TEST-ID-001..002) | Test execution | REQ-SR-ID-001..002 | All |
| EVI-VER-004 | Fault injection test results (FI-001..004) | Test execution | REQ-SR-PHY-001..002 | ISO 21434, IEC 62443 |
| EVI-VER-005 | Timing measurement report (TIM-001..002) | Measurement | REQ-SR-SCH-001 | All |
| EVI-VER-006 | TVLA test results | Measurement | REQ-SR-SCH-001 | ISO 21434, IEC 62443 |
| EVI-VER-007 | Fuzzing campaign report | Fuzzing | REQ-FR-BOOT-001 | All |
| EVI-VER-008 | Red team test report | Red team execution | All REQ-SR | ISO 21434 §9.6 |
| EVI-VER-009 | Coverage report | Coverage measurement | All | DO-178C, ISO 26262-6 |
| EVI-VER-010 | Traceability matrix | `Traceability_Matrix.md` + test results | All | All |
| EVI-VER-011 | Formal verification results | Model checker / CBMC output | REQ-SR-BOOT-001 | DO-178C DO-333 |
| EVI-VER-012 | Physical pentest report | Third-party | REQ-SR-PHY-001..002 | IEC 62443 |

### 2.4 Operational Evidence (EVI-OPS)

| ID | Evidence Item | Source | Requirements Addressed | Standards |
| --- | --- | --- | --- | --- |
| EVI-OPS-001 | Key generation ceremony records | Ceremony logs | REQ-SR-KEY-001, REQ-SR-SC-001 | All |
| EVI-OPS-002 | Signing operation audit logs | HSM audit trail | REQ-SR-KEY-001 | ISO 21434 §9.7 |
| EVI-OPS-003 | Device provisioning records | Backend logs | REQ-SR-MFR-001..002 | IEC 62443 |
| EVI-OPS-004 | OTA deployment records | Backend logs | REQ-SR-OTA-001..003 | ISO 21434 |
| EVI-OPS-005 | Incident response records | IR documentation | — | ISO 21434 §9.4, IEC 62443 |
| EVI-OPS-006 | Monitoring and telemetry data | Backend dashboard | — | IEC 62443 |
| EVI-OPS-007 | CVE monitoring log | Dependency scanner | REQ-SR-SC-001 | IEC 62304 §5.3 |
| EVI-OPS-008 | Audit trail (all standards) | `Audit_Trace.md` + evidence | All | All |

### 2.5 Process Evidence (EVI-PRO)

| ID | Evidence Item | Source | Requirements Addressed | Standards |
| --- | --- | --- | --- | --- |
| EVI-PRO-001 | Design review records | Review log | All | ISO 21434 §9.3 |
| EVI-PRO-002 | Security review records | Review log | All | IEC 62443 |
| EVI-PRO-003 | Risk acceptance sign-offs | `Residual_Risk.md` | — | ISO 21434 §9.5 |
| EVI-PRO-004 | Supply chain audit reports | Audit records | REQ-SR-SC-001 | ISO 21434 §10 |
| EVI-PRO-005 | Factory audit reports | Audit records | REQ-SR-MFR-001..002 | IEC 62443 |
| EVI-PRO-006 | Tool qualification records | Qualification reports | — | DO-178C DO-330, IEC 61508-3 |
| EVI-PRO-007 | Annual security review report | Review document | All | All |

---

## 3. Evidence Mapping by Standard

### 3.1 ISO 21434 Evidence

| Clause | Evidence Items |
| --- | --- |
| §9.3 (Design) | EVI-DES-001, EVI-DES-007, EVI-DES-008, EVI-DES-009, EVI-PRO-001 |
| §9.4 (Integration & Verification) | EVI-VER-001..012, EVI-OPS-005 |
| §9.5 (TARA) | EVI-DES-003, EVI-DES-004, EVI-PRO-003 |
| §9.6 (Validation) | EVI-VER-008 |
| §9.7 (Production) | EVI-DES-006, EVI-OPS-001, EVI-OPS-002 |
| §9.8 (Operations) | EVI-IMP-005, EVI-OPS-003..008 |
| §10 (Supply Chain) | EVI-IMP-004, EVI-IMP-006, EVI-PRO-004 |

### 3.2 IEC 62443 Evidence

| CR | Evidence Items |
| --- | --- |
| CR 1.1 (Human user identification) | EVI-DES-002 |
| CR 1.3 (Account management) | EVI-DES-006 |
| CR 2.1 (Authorization enforcement) | EVI-DES-005, EVI-VER-012 |
| CR 3.1 (Communication integrity) | EVI-DES-003, EVI-VER-004 |
| CR 3.2 (Malicious code protection) | EVI-VER-001, EVI-VER-002 |
| CR 4.1 (Information confidentiality) | EVI-VER-005, EVI-VER-006 |

### 3.3 DO-178C Evidence (DAL A)

| Objective | Evidence Items |
| --- | --- |
| Software development plan (PSAC) | EVI-DES-010 |
| Software requirements data | EVI-DES-001, EVI-DES-007 |
| Design description | EVI-DES-001, EVI-DES-009 |
| Source code | EVI-IMP-001..003 |
| Executable object code | Build output + EVI-IMP-006 |
| Software verification results | EVI-VER-001..012 |
| Software configuration index | SBOM (EVI-IMP-005) |
| Software accomplishment summary | Audit report |

---

## 4. Evidence Quality Requirements

| Property | Requirement |
| --- | --- |
| Authenticity | Evidence must be attributable to its source (digital signature, audit log) |
| Integrity | Evidence must be tamper-evident (hash, signature, write-once storage) |
| Traceability | Every evidence item must trace to ≥ 1 requirement and ≥ 1 standard clause |
| Currency | Evidence must be regenerated when relevant system component changes |
| Reproducibility | Test evidence must be reproducible (known input → known output) |
| Completeness | No requirement without evidence; no standard clause without evidence |
| Retention | Retained per retention schedule in Review_and_Audit_Process.md §8.2 |

---

## 5. Evidence Storage and Access

### 5.1 Repository

```
Evidence Repository
├── design/
│   ├── architecture/
│   ├── security/
│   └── contracts/
├── implementation/
│   ├── source/
│   ├── build/
│   └── sbom/
├── verification/
│   ├── tests/
│   ├── fault_injection/
│   ├── side_channel/
│   └── red_team/
├── operations/
│   ├── key_management/
│   ├── provisioning/
│   └── incidents/
└── process/
    ├── reviews/
    ├── audits/
    └── risk_acceptance/
```

### 5.2 Access Control

| Role | Access Level |
| --- | --- |
| Security Architect | Read/Write all |
| Development Team | Read all, Write implementation/verification |
| QA Team | Read all, Write verification |
| Auditor (external) | Read-only (time-limited, scoped) |
| Product Manager | Read all |
| Others | No access by default |

---

## 6. Evidence Lifecycle

```
Create → Review → Approve → Store → Maintain → Archive → Destroy
  │        │        │         │        │          │         │
  │    Peer       Sign-off   Secure   Update     After      Per
  │    review     by owner   repo     on change  retention  policy
```

---

## 7. References

| Document | Reference |
| --- | --- |
| Audit Trace | `Audit_Trace.md` |
| Work Products | `Work_Products.md` |
| Review and Audit Process | `Review_and_Audit_Process.md` |
| Compliance Overview | `Compliance_Overview.md` |
