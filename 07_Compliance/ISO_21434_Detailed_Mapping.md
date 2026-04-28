# ISO/SAE 21434 Detailed Compliance Mapping

**Document ID:** CMP-ISO21434-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document provides the detailed clause-by-clause mapping between the SBOP platform and ISO/SAE 21434:2021 *Road vehicles -- Cybersecurity engineering*.

It demonstrates how each SBOP artifact contributes to satisfying the corresponding ISO 21434 requirement and identifies gaps requiring additional work products.

---

## 2. Reference Architecture

SBOP targets a **Tier-1 component** role within the ISO 21434 framework. The item definition (Clause 9.3) for SBOP is:

> Secure Boot and OTA Update Platform -- a hardware-agnostic security framework providing firmware authenticity, integrity, anti-rollback, device identity, and OTA lifecycle management for embedded automotive ECUs.

---

## 3. Clause 5: Organizational Cybersecurity Management

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 5.2 | Cybersecurity policy | → `00_Architecture/Security_Overview.md` | Security objectives, principles |
| 5.3 | Cybersecurity governance | → `08_Product/Reference_Architecture.md` | Defined roles and responsibilities |
| 5.4 | Cybersecurity culture | → `07_Compliance/Review_and_Audit_Process.md` | Continuous review process |
| 5.5 | Information sharing | → `06_Operations/Incident_Response.md` | Incident communication procedures |
| 5.6 | Management system | → `07_Compliance/Work_Products.md` | Work product tracking |
| 5.7 | Tool management | → `04_Security/Supply_Chain_Security.md` (Phase 3.1) | Toolchain integrity |
| 5.8 | Audit | → `07_Compliance/Audit_Trace.md` | Audit traceability |

---

## 4. Clause 6: Project-dependent Cybersecurity Management

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 6.2 | Cybersecurity responsibilities | → `00_Architecture/System_Overview.md` (Owner field) | Role assignment per document |
| 6.3 | Cybersecurity plan | → `07_Compliance/Compliance_Overview.md` | Compliance strategy defined |
| 6.4 | Tailoring | → `00_Architecture/System_Overview.md` (Section 3 - Scope) | Scope and non-goals documented |
| 6.5 | Reuse | → `02_System_Design/System_Decomposition.md` | Component abstraction for reuse |
| 6.6 | Off-the-shelf component | → `07_Compliance/Medical_Compliance_Mapping.md` (SOUP section) | Third-party dependency assessment |
| 6.7 | Cybersecurity case | → `04_Security/Security_Case.md` | Complete security argument |

---

## 5. Clause 7: Distributed Cybersecurity Activities

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 7.2 | Supplier request | → `04_Security/Manufacturing_Security.md` (Phase 3.5) | Supplier security requirements |
| 7.3 | Supplier evaluation | → `04_Security/Manufacturing_Security.md` | Factory audit requirements |
| 7.4 | Supplier agreement | → `06_Operations/Provisioning_Operations.md` | Manufacturing interface |
| 7.5 | Development interface | → `02_System_Design/System_Context.md` | External entity boundaries |

---

## 6. Clause 8: Continuous Cybersecurity Activities

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 8.2 | Cybersecurity monitoring | → `06_Operations/Monitoring_and_Telemetry.md` | Runtime observability |
| 8.3 | Cybersecurity event evaluation | → `06_Operations/Incident_Response.md` | Assessment procedures |
| 8.4 | Vulnerability analysis | → `04_Security/Attack_Tree.md` | Structured attack analysis |
| 8.5 | Vulnerability management | → `06_Operations/OTA_Deployment.md` | Update deployment for fixes |
| 8.6 | Cybersecurity assessment | → `07_Compliance/Review_and_Audit_Process.md` | Internal/external review |
| 8.7 | Information sharing | → `06_Operations/Incident_Response.md` (Section 5) | Stakeholder notification |

---

## 7. Clause 9: Concept Phase

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 9.2 | Item definition | → `00_Architecture/System_Overview.md` + `System_Context.md` | System scope and context |
| 9.3 | Cybersecurity goals | → `04_Security/Security_Goals.md` | Authenticity, Integrity, Anti-Cloning |
| 9.4 | Cybersecurity concept | → `00_Architecture/Security_Overview.md` | Security domains and controls |
| 9.5 | **TARA** | → `04_Security/TARA_Methodology.md` (Phase 2.2) | **Formal TARA document** |

### 7.1 TARA Requirements per Clause 9.5

| TARA Activity | SBOP Artifact | Status |
| --- | --- | --- |
| Asset identification | `TARA_Methodology.md` Section 3 | Defined |
| Threat scenario identification | `Attack_Tree.md` | Defined |
| Impact rating (S, F, O, P) | `TARA_Methodology.md` Section 5 | Defined |
| Attack path analysis | `Attack_Tree.md` + `TARA_Methodology.md` Section 6 | Defined |
| Attack feasibility rating | `Risk_Quantification.md` (CVSS) | Defined |
| Risk determination | `Risk_Quantification.md` + TARA risk matrix | Defined |
| Risk treatment decision | `Risk_Treatment.md` | Defined |

---

## 8. Clause 10: Product Development

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 10.2 | Design specification | → `02_System_Design/` (all docs) | Complete design specification |
| 10.3 | Cybersecurity specification | → `01_Requirements/REQ-SR-SECURITY.md` | Security requirements |
| 10.4.1 | Integration specification | → `02_System_Design/System_Decomposition.md` | Component integration |
| 10.4.2 | Verification specification | → `05_Verification/Test_Strategy.md` | Verification approach |
| 10.4.3 | Cybersecurity-related items | → `03_Subsystem/` (all docs) | Subsystem specifications |
| 10.5 | Verification | → `05_Verification/Test_Cases.md` | Concrete test cases |
| 10.6 | Vulnerability analysis | → `04_Security/Attack_Tree.md` + `Residual_Risk.md` | Post-design analysis |

---

## 9. Clause 11: Cybersecurity Validation

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 11.2 | Validation plan | → `05_Verification/Test_Strategy.md` | Validation approach |
| 11.3 | Validation specification | → `05_Verification/Red_Team_Test_Plan.md` | Attack simulation |
| 11.4 | Validation execution | → `05_Verification/` test results | Test execution records |

---

## 10. Clause 12: Production

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 12.2 | Production cybersecurity | → `06_Operations/Provisioning_Operations.md` | Factory/provisioning flow |
| 12.3 | Production control plan | → `04_Security/Manufacturing_Security.md` (Phase 3.5) | Manufacturing controls |
| 12.4 | Post-production | → `06_Operations/OTA_Deployment.md` | Post-deployment update |

---

## 11. Clause 13: Operations and Maintenance

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 13.2 | Incident response | → `06_Operations/Incident_Response.md` | Full IR plan |
| 13.3 | Remedial action | → `06_Operations/OTA_Deployment.md` | OTA deployment/rollback |
| 13.4 | Update | → `03_Subsystem/Update/OTA_Overview.md` | Update architecture |

---

## 12. Clause 14: End of Cybersecurity Support

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 14.2 | End of support | → `03_Subsystem/Identity/Identity_Overview.md` | Lifecycle: Operational → Revoked |
| 14.3 | Communication | → `06_Operations/Incident_Response.md` | Stakeholder communication |
| 14.4 | Decommissioning procedure | → `03_Subsystem/Identity/Provisioning_Flow.md` | Lock state handling |

---

## 13. Clause 15: Threat Analysis and Risk Assessment Methods

| ISO 21434 Clause | Description | SBOP Mapping | Evidence |
| --- | --- | --- | --- |
| 15.2 | TARA methods | → `04_Security/TARA_Methodology.md` | Formal methodology |
| 15.3 | Asset identification | → TARA Section 3 | Asset inventory |
| 15.4 | Threat scenarios | → `04_Security/Attack_Tree.md` | Structured attack paths |
| 15.5 | Impact rating | → TARA Section 5 | S, F, O, P dimensions |
| 15.6 | Attack feasibility | → `04_Security/Risk_Quantification.md` | L × I model + CVSS |
| 15.7 | Risk value | → `04_Security/Risk_Quantification.md` (Risk Table) | Risk levels |
| 15.8 | Risk treatment | → `04_Security/Risk_Treatment.md` | Treatment decisions |

---

## 14. Consolidated Evidence Matrix

| ISO 21434 Clause | SBOP Document | Work Product ID | Status |
| --- | --- | --- | --- |
| 5.2 Cybersecurity policy | `Security_Overview.md` | WP-01 | Complete |
| 5.7 Tool management | `Supply_Chain_Security.md` | WP-07 | Planned (Phase 3.1) |
| 5.8 Audit | `Audit_Trace.md` | WP-05 | Partial |
| 6.7 Security case | `Security_Case.md` | WP-03 | Complete |
| 7.3 Supplier evaluation | `Manufacturing_Security.md` | WP-11 | Planned (Phase 3.5) |
| 8.4 Vulnerability analysis | `Attack_Tree.md` | WP-03 | Complete |
| 9.5 TARA | `TARA_Methodology.md` | WP-04 | Defined |
| 10.2 Design | `02_System_Design/` | WP-02 | Complete |
| 10.4.1 Integration | `System_Decomposition.md` | WP-02 | Complete |
| 10.5 Verification | `Test_Strategy.md` + `Test_Cases.md` | WP-06 | Partial |
| 11.4 Validation | `Red_Team_Test_Plan.md` | WP-06 | Defined |
| 12.2 Production | `Provisioning_Operations.md` | WP-08 | Complete |
| 13.2 IR | `Incident_Response.md` | WP-09 | Complete |
| 13.4 Update | `OTA_Overview.md` | WP-10 | Complete |
| 15.2 TARA methods | `TARA_Methodology.md` | WP-04 | Defined |

---

## 15. Gap Analysis

| Gap ID | Description | Impact | Mitigation |
| --- | --- | --- | --- |
| GAP-21434-001 | No formal CAL (Cybersecurity Assurance Level) assignment per item | Medium | → `CAL_SLT_Definitions.md` (Phase 2.5) |
| GAP-21434-002 | No documented cybersecurity validation results (only plans) | Medium | Execute `Red_Team_Test_Plan.md` |
| GAP-21434-003 | Manufacturing security not yet detailed | Low | → `Manufacturing_Security.md` (Phase 3.5) |
| GAP-21434-004 | Supply chain tool management not yet detailed | Low | → `Supply_Chain_Security.md` (Phase 3.1) |
| GAP-21434-005 | No post-development monitoring evidence (spec only) | Low | Operational data collection required |
