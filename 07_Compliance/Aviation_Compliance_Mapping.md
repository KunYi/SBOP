# Aviation Standards Compliance Mapping (DO-178C / DO-254)

**Document ID:** CMP-AV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document maps the SBOP platform to aviation software and hardware development assurance standards:

- **DO-178C**: Software Considerations in Airborne Systems and Equipment Certification
- **DO-254**: Design Assurance Guidance for Airborne Electronic Hardware
- **ARP4754A**: Guidelines for Development of Civil Aircraft and Systems
- **DO-326A / ED-202A**: Airworthiness Security Process Specification

---

## 2. Aviation Domain Context

### 2.1 Applicability

SBOP is applicable to aviation systems as an **airborne software component** installed on avionics Line Replaceable Units (LRUs):

- Flight control computers
- Engine control units (FADEC)
- Cabin and navigation systems
- Communication and surveillance systems
- In-flight entertainment (non-critical)

### 2.2 Aviation-Specific Considerations

| Constraint | Impact on SBOP | Mitigation |
| --- | --- | --- |
| DO-178C software level (A-E) | Verification rigor increases with level | Level-based process adaptation |
| Civil certification authority oversight | All artifacts subject to audit | → Traceability from plan |
| Airworthiness security (DO-326A) | Cybersecurity certification for aircraft systems | → Security_Case alignment |
| Dual-channel / dissimilar redundancy | Bootloader may need independent dissimilar implementation | Architectural note |
| Extended operating conditions | -55°C to +125°C ambient for some installations | Platform integration note |

---

## 3. DO-178C Software Levels

### 3.1 Software Level Definitions

| Level | Failure Condition | Description | SBOP Applicability |
| --- | --- | --- | --- |
| **Level A** | Catastrophic | Failure may cause multiple fatalities; hull loss | Flight-critical avionics boot |
| **Level B** | Hazardous / Severe-Major | Failure may cause serious or fatal injuries to small number | Navigation, engine control |
| **Level C** | Major | Failure may cause discomfort or injuries | Cabin systems, communications |
| **Level D** | Minor | Failure may cause inconvenience | Non-critical avionics |
| **Level E** | No Safety Effect | Failure has no safety impact | In-flight entertainment |

SBOP targets **Level A** capability, recognizing that the same boot mechanism may be deployed across all levels and must not be the weakest link.

### 3.2 Verification Objectives by Level

| DO-178C Objective | Level A | Level B | Level C | Level D |
| --- | --- | --- | --- | --- |
| Requirements traceable to system | ✓ | ✓ | ✓ | ✓ |
| Code traceable to requirements | ✓ | ✓ | ✓ | — |
| Structural coverage: Statement | ✓ | ✓ | — | — |
| Structural coverage: Decision | ✓ | ✓ | — | — |
| Structural coverage: MC/DC | ✓ | — | — | — |
| Requirements-based testing | ✓ | ✓ | ✓ | ✓ |
| Robustness testing | ✓ | ✓ | — | — |
| Stack analysis | ✓ | ✓ | ✓ | — |
| Worst-case execution time | ✓ | ✓ | — | — |

---

## 4. DO-178C Process Mapping

### 4.1 Planning Process (DO-178C Section 4)

| DO-178C Objective | SBOP Artifact | Evidence |
| --- | --- | --- |
| PSAC (Plan for SW Aspects of Certification) | → `07_Compliance/Compliance_Overview.md` + `Work_Products.md` | Master compliance plan |
| SDP (Software Development Plan) | → `08_Product/Reference_Architecture.md` | Development process |
| SVP (Software Verification Plan) | → `05_Verification/Test_Strategy.md` | Verification approach |
| SCMP (Software Configuration Management Plan) | → `07_Compliance/Work_Products.md` (version control) | Configuration management |
| SQAP (Software Quality Assurance Plan) | → `07_Compliance/Review_and_Audit_Process.md` | Quality assurance |

### 4.2 Development Process (DO-178C Section 5)

| DO-178C Objective | SBOP Artifact | Evidence |
| --- | --- | --- |
| High-Level Requirements (HLR) | → `01_Requirements/` (complete REQ set) | Requirements docs |
| Derived High-Level Requirements | → Requirements traceable to system safety analysis | Traceability matrix |
| Software Architecture | → `02_System_Design/System_Decomposition.md` | Architecture docs |
| Low-Level Requirements (LLR) | → `03_Subsystem/` (subsystem detailed docs) | Subsystem specs |
| Source Code | Not applicable (Tier-1 spec) | — |
| Executable Object Code | Not applicable (Tier-1 spec) | — |

### 4.3 Verification Process (DO-178C Section 6)

| DO-178C Objective | SBOP Artifact | Evidence |
| --- | --- | --- |
| Reviews of HLR | → `07_Compliance/Review_and_Audit_Process.md` | Review records |
| Reviews of Architecture | → Design review process | Review records |
| Reviews of Source Code | Not applicable (Tier-1) | — |
| Requirements-Based Testing | → `05_Verification/Test_Cases.md` | Test cases traceable to REQs |
| Robustness Testing | → `05_Verification/Fault_Injection_Test.md` + `Fuzzing_Strategy.md` | Robustness evidence |
| Structural Coverage Analysis | → `05_Verification/Coverage_Model.md` | Coverage reports |
| Requirements Traceability | → `05_Verification/Traceability_Matrix.md` | Bidirectional trace |

### 4.4 Configuration Management (DO-178C Section 7)

| DO-178C Objective | SBOP Artifact | Evidence |
| --- | --- | --- |
| Problem Reporting | → `06_Operations/Incident_Response.md` | IR process |
| Change Control | Deterministic build (REQ-NFR-SYS-003) | Build reproducibility |
| Configuration Status Accounting | → `07_Compliance/Work_Products.md` | Version tracking |
| Archive and Retrieval | → `07_Compliance/Evidence_List.md` (Storage) | Evidence archival |

### 4.5 Quality Assurance (DO-178C Section 8)

| DO-178C Objective | SBOP Artifact | Evidence |
| --- | --- | --- |
| QA reviews of plans | → `Review_and_Audit_Process.md` | Review records |
| QA reviews of development | → Design review process | Review records |
| QA reviews of verification | → `05_Verification/Coverage_Model.md` | Coverage validation |
| QA conformity review | → `07_Compliance/Evidence_List.md` | Evidence package |

---

## 5. DO-254 Hardware Mapping

SBOP interacts with avionics hardware (FPGA, ASIC, PLD) that may be subject to DO-254:

| DO-254 Consideration | SBOP Interaction |
| --- | --- |
| Hardware design assurance level (A-D) | Matches software level of the LRU |
| Hardware/software interface | → `03_Subsystem/Boot/Boot_Interface.md` (storage, crypto) |
| Fused/immutable boot ROM | Hardware must provide immutable initial boot |
| Secure element certification | If used for key storage: must be assessed per DO-254 |
| COTS IP assessment | Crypto accelerator IP must be assessed for design assurance |

---

## 6. DO-326A / ED-202A Airworthiness Security

### 6.1 Security Process Mapping

DO-326A defines an airworthiness security process for aircraft systems:

| DO-326A Activity | SBOP Mapping |
| --- | --- |
| Security scope definition | → `00_Architecture/System_Overview.md` (Section 3) |
| Security risk assessment | → `04_Security/TARA_Methodology.md` + `Risk_Quantification.md` |
| Security requirements | → `01_Requirements/REQ-SR-SECURITY.md` |
| Security architecture | → `02_System_Design/` (all) |
| Security verification | → `05_Verification/` |
| Security assurance | → `04_Security/Security_Case.md` |

### 6.2 Security Effectiveness Assurance Level (SEAL)

| SEAL | Threats Addressed | SBOP Mapping |
| --- | --- | --- |
| SEAL 1 | Basic attacker | SL 1 controls |
| SEAL 2 | Enhanced attacker | SL 2 controls |
| SEAL 3 | Sophisticated attacker | SL 3 controls |
| SEAL 4 | Advanced persistent attacker | SL 4 controls |

SBOP targets **SEAL 3** for Level A/B systems, **SEAL 2** for Level C/D.

---

## 7. CAST-32A / A(M)C 20-193 Considerations

### 7.1 Multi-Core Processor Certification

If SBOP is deployed on a multi-core processor (MCP) in avionics:

| CAST-32A Objective | SBOP Impact |
| --- | --- |
| MCP resource interference | Boot must operate on dedicated core or during single-core init phase |
| Shared resource verification | Verify that crypto operations don't interfere with other cores |
| Deterministic execution | Boot sequence must be deterministic regardless of other core activity |

### 7.2 Mitigation

- SBOP boot must complete before multi-core SMP is enabled
- Boot must run on a locked single core with interrupts disabled

---

## 8. ARP4754A System Development

| ARP4754A Process | SBOP Mapping |
| --- | --- |
| Aircraft function development | System scope per `System_Overview.md` |
| Allocation to items | SBOP as item within LRU |
| System safety assessment | → Safety analysis (Functional Hazard Assessment) |
| Development assurance level assignment | → This document (Section 3) |
| Validation and verification | → `05_Verification/` |

---

## 9. Design Assurance Considerations

### 9.1 Level A Boot Verification Example

For a Level A deployment, the boot verification logic must achieve MC/DC (Modified Condition/Decision Coverage):

| Condition | Decision | Test Case |
| --- | --- | --- |
| Signature valid / invalid | Boot / Fail-safe | TEST-BOOT-002 (valid), TEST-BOOT-002 (invalid) |
| Hash match / mismatch | Boot / Fail-safe | TEST-BOOT-001 (match), TEST-BOOT-003 (mismatch) |
| Version OK / rollback | Boot / Fail-safe | TEST-BOOT-001 (OK), TEST-BOOT-004 (rollback) |

For MC/DC at Level A, each condition must independently affect the decision, requiring the following test vectors:

```
signature_valid = {true, false}  ×  hash_match = {true, false}  ×  version_ok = {true, false}

Required test cases:
1. {T, T, T} → Boot (all pass)
2. {F, T, T} → Fail-safe (signature independently causes failure)
3. {T, F, T} → Fail-safe (hash independently causes failure)
4. {T, T, F} → Fail-safe (version independently causes failure)
5. {F, F, F} → Fail-safe (any combination of failures)

Note: {F, T, T} and {T, F, T} can never be reached simultaneously
because signature failure terminates before hash check. MC/DC for the
early-return structure must be demonstrated at the architecture level.
```

### 9.2 Level A Independence Requirements

For Level A systems, DO-178C requires independence between development and verification. SBOP's specification:

| Role | Description |
| --- | --- |
| Development | Authors of SBOP requirement, design, and implementation artifacts |
| Verification | Independent review and testing of SBOP artifacts; separate team |
| Certification Liaison | Interface with certification authority (FAA/EASA) |

---

## 10. Aviation Evidence Package

### 10.1 Required DO-178C Life Cycle Data

| Data Item | SBOP Equivalent | Level A | Level C |
| --- | --- | --- | --- |
| PSAC | `Compliance_Overview.md` | Required | Required |
| SDP | `Reference_Architecture.md` | Required | Required |
| SVP | `Test_Strategy.md` | Required | Required |
| SCMP | `Work_Products.md` | Required | Required |
| SQAP | `Review_and_Audit_Process.md` | Required | Required |
| Software Requirements Data | `01_Requirements/` | Required | Required |
| Design Description | `02_System_Design/` + `03_Subsystem/` | Required | Required |
| Source Code | Not applicable (Tier-1) | Required | Required |
| Executable Object Code | Not applicable (Tier-1) | Required | Required |
| Software Verification Cases & Procedures | `05_Verification/` | Required | Required |
| Software Verification Results | Test execution results | Required | Required |
| Software Life Cycle Environment Config Index | Build environment description | Required | Required |
| Software Configuration Index | Version manifest per build | Required | Required |
| Problem Reports | `Incident_Response.md` records | Required | Required |
| Software Completion Summary | Compliance summary | Required | Required |

---

## 11. Gap Analysis (Aviation Domain)

| Gap ID | Description | Impact | Priority |
| --- | --- | --- | --- |
| GAP-AV-001 | No formal PSAC (Plan for Software Aspects of Certification) | Required for certification submission | P0 |
| GAP-AV-002 | No structural coverage analysis for Level A (MC/DC) | Required for Level A verification | P0 |
| GAP-AV-003 | No worst-case execution time (WCET) analysis for boot phases | Required for all levels | P1 |
| GAP-AV-004 | No stack analysis for boot execution | Required for Level A-C | P1 |
| GAP-AV-005 | No independence plan for development vs. verification | Required for Level A/B | P1 |
| GAP-AV-006 | No tool qualification records (compiler, linker, analysis tools) | Required if tools used for verification | P2 |
| GAP-AV-007 | No CAST-32A multi-core interference analysis | Required if deployed on MCP | P2 |
| GAP-AV-008 | No dissimilar redundancy analysis for dual-channel systems | Required for Level A dual-channel | P2 |

---

## 12. Commitment to Next Phase (Reference Implementation)

The aviation compliance mapping is a Tier-1 specification artifact. Full DO-178C certification evidence requires:

1. **Implementation phase**: Source code, build environment, tool qualification
2. **Verification phase**: Test execution, coverage measurement, WCET analysis
3. **Certification phase**: PSAC approval, SOI audits, final compliance statement

→ These will be addressed in the reference platform implementation phase.

---

## 13. References

- RTCA DO-178C: Software Considerations in Airborne Systems and Equipment Certification
- RTCA DO-254: Design Assurance Guidance for Airborne Electronic Hardware
- RTCA DO-326A: Airworthiness Security Process Specification
- SAE ARP4754A: Guidelines for Development of Civil Aircraft and Systems
- FAA AC 20-115D: RTCA DO-178C recognized guidance
- EASA CM-SWCEH-002: Software Aspects of Certification
- CAST-32A: Multi-core Processors Position Paper
