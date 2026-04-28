# Work Products Definition

**Document ID:** CMP-WP-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines required documents aligned with compliance standards across all 6 standards.

---

## 2. Work Products by Phase

### 2.1 Concept Phase

| Work Product | Standards | SBOP Document |
|--------------|-----------|---------------|
| System Context | ISO 21434 §10, IEC 62443, DO-326A | `System_Context.md` |
| Threat Model | ISO 21434 §9.5, ISO 14971 §4 | `Threat_Model.md` |
| TARA / Risk Assessment | ISO 21434 §9.5, ISO 14971 §4.4 | `TARA_Methodology.md`, `Risk_Quantification.md` |
| Security Goals | ISO 21434 §10, IEC 62443 | `Security_Goals.md` |
| Security Concept | ISO 21434 §10 | `Security_Case.md`, `Security_Overview.md` |

### 2.2 Design Phase

| Work Product | Standards | SBOP Document |
|--------------|-----------|---------------|
| System Architecture | IEC 62304 §5.3, ECSS E-ST-40 | `System_Overview.md`, `System_Decomposition.md` |
| Software Architecture | IEC 62304 §5.4, DO-178C | `System_Decomposition.md`, subsystem docs |
| Interface Design | IEC 62304 §5.5, DO-178C | `Interface_Contracts.md`, subsystem interface docs |
| Security Architecture | ISO 21434 §10, IEC 62443 | `Security_Case.md`, `Zone_Conduit_Model.md` |
| Key Management | ISO 21434, IEC 62443 CR 6.1 | `Key_Hierarchy.md`, `Key_Derivation.md` |
| Trust Model | ISO 21434, ECSS Q-ST-80 | `Trust_Model.md` |
| State Machine Design | IEC 62304 §5.4, DO-178C | `State_Machine.md`, `Unified_State_Model.md` |

### 2.3 Implementation Phase

| Work Product | Standards | SBOP Document |
|--------------|-----------|---------------|
| Crypto Algorithm Spec | IEC 62443, FIPS 140-2 | `Crypto_Algorithms.md` |
| Error Handling Design | IEC 62304 §5.6, DO-178C | `Error_Code_Catalog.md`, failure model docs |
| Supply Chain Security | ISO 21434 §12, DO-326A | `Supply_Chain_Security.md` |
| Manufacturing Security | ISO 21434 §12 | `Manufacturing_Security.md` |
| Physical Design | ISO 21434, IEC 62443 CR 7.1 | `Physical_Tamper_Resistance.md` |
| Side-Channel Design | ISO 21434, IEC 62443 | `Side_Channel_Countermeasures.md` |
| Debug Architecture | ISO 21434, IEC 62443 | `Secure_Debug_Architecture.md` |

### 2.4 Verification Phase

| Work Product | Standards | SBOP Document |
|--------------|-----------|---------------|
| Test Strategy | IEC 62304 §5.7, DO-178C | `Test_Strategy.md` |
| Test Cases | All standards | `Test_Cases.md` |
| Traceability Matrix | All standards | `Traceability_Matrix.md` |
| Audit Trace | All standards | `Audit_Trace.md` |
| Fault Injection Plan | ISO 21434, IEC 61508, ISO 26262 | `Fault_Injection_Test.md` |
| Fuzzing Strategy | IEC 62443 CR 3.2 | `Fuzzing_Strategy.md` |
| Red Team Test Plan | ISO 21434, IEC 62443 | `Red_Team_Test_Plan.md` |
| Coverage Model | DO-178C, IEC 61508 | `Coverage_Model.md` |
| Compliance Mapping | All standards | `Compliance_Mapping.md` |

### 2.5 Operations Phase

| Work Product | Standards | SBOP Document |
|--------------|-----------|---------------|
| Incident Response Plan | ISO 21434 §13 | (To be defined per deployment) |
| Key Ceremony Procedure | ISO 21434, IEC 62443 | (Operations manual) |
| Monitoring & Detection | ISO 21434 §13 | Anomaly detection strategy |
| Decommissioning Procedure | ISO 21434 §14 | (Operations manual) |

---

## 3. Standards Coverage Matrix

| Standard | Concept | Design | Implementation | Verification | Operations |
|----------|---------|--------|----------------|--------------|------------|
| ISO/SAE 21434 | 5 docs | 7 docs | 7 docs | 9 docs | 4 docs |
| IEC 62443 | 5 docs | 6 docs | 6 docs | 9 docs | 3 docs |
| ECSS | 4 docs | 5 docs | 4 docs | 7 docs | 2 docs |
| IEC 62304 / ISO 14971 | 5 docs | 6 docs | 6 docs | 9 docs | 3 docs |
| DO-178C / DO-254 / DO-326A | 3 docs | 6 docs | 5 docs | 9 docs | 2 docs |
| IEC 61508 / ISO 26262 | 4 docs | 5 docs | 5 docs | 8 docs | 2 docs |

---

## 4. Document Status Tracking

| Status | Meaning |
|--------|---------|
| Draft | Initial version, under development |
| Review | Peer review in progress |
| Approved | Approved for use |
| Deprecated | Superseded by newer version |
| Retired | No longer applicable |

---

## 5. References

| Document | Reference |
|----------|-----------|
| Compliance Overview | `Compliance_Overview.md` |
| Compliance Mapping | `Compliance_Mapping.md` |
| Audit Trace | `Audit_Trace.md` |
