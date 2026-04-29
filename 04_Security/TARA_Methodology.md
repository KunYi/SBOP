# TARA Methodology (ISO 21434 Compliant)

**Document ID:** SEC-TARA-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the Threat Analysis and Risk Assessment (TARA) methodology for the SBOP platform, aligned with ISO/SAE 21434:2021 Clause 9.5 and Clause 15.

The TARA methodology is the formal process by which SBOP identifies assets, analyzes threats, evaluates risks, and determines risk treatment.

---

## 2. TARA Process Overview

The TARA process follows the ISO 21434 prescribed sequence:

```
Step 1: Asset Identification
    ↓
Step 2: Threat Scenario Identification
    ↓
Step 3: Impact Rating (S, F, O, P)
    ↓
Step 4: Attack Path Analysis
    ↓
Step 5: Attack Feasibility Rating
    ↓
Step 6: Risk Determination
    ↓
Step 7: Risk Treatment Decision
```

Each step produces a defined work product and feeds into the next. Iterations may occur as new threats or assets are discovered during later phases.

---

## 3. Asset Inventory

### 3.1 Asset Categories

| Asset ID | Asset Name | Category | Security Property | Damage Scenario |
| --- | --- | --- | --- | --- |
| AS-001 | Firmware Image | Software | Integrity, Authenticity | Unauthorized execution → loss of vehicle control |
| AS-002 | Bootloader | Software | Integrity | Bootloader compromise → persistent attacker control |
| AS-003 | Device Identity (UID) | Data | Authenticity | Identity cloning → fleet compromise |
| AS-004 | Root Key (KR) | Key Material | Confidentiality, Integrity | Key extraction → signing capability for attacker |
| AS-005 | Device Key (KD) | Key Material | Confidentiality | Key extraction → device impersonation |
| AS-006 | Firmware Signing Key | Key Material | Confidentiality, Integrity | Key compromise → arbitrary firmware signing |
| AS-007 | Boot State Machine | Logic/Process | Availability, Integrity | State bypass → execution without verification |
| AS-008 | OTA Update Process | Logic/Process | Integrity, Availability | Update corruption → device bricking |
| AS-009 | Provisioning Data | Data | Confidentiality, Integrity | Provisioning data leak → device cloning |
| AS-010 | Version Counter | Data | Integrity | Rollback → known-vulnerability exploitation |

### 3.2 Asset Criticality

Each asset is classified against the ISO 21434 damage scenario dimensions:

| Asset ID | Safety (S) | Financial (F) | Operational (O) | Privacy (P) |
| --- | --- | --- | --- | --- |
| AS-001 | Critical | Major | Critical | Negligible |
| AS-002 | Critical | Major | Critical | Negligible |
| AS-003 | Negligible | Major | Major | Moderate |
| AS-004 | Critical | Critical | Critical | Negligible |
| AS-005 | Major | Major | Major | Moderate |
| AS-006 | Critical | Critical | Critical | Negligible |
| AS-007 | Critical | Major | Critical | Negligible |
| AS-008 | Major | Major | Major | Negligible |
| AS-009 | Negligible | Major | Moderate | Major |
| AS-010 | Major | Moderate | Major | Negligible |

---

## 4. Threat Scenario Catalog

### 4.1 Threat Scenario Identification

Threat scenarios are identified through:

* Attack tree analysis (→ `Attack_Tree.md`)
* Vulnerability analysis (→ `Attack_Surface.md`)
* Historical vulnerability analysis
* Expert review

### 4.2 Threat Scenario Table

| Threat ID | Threat Scenario | Target Asset | Threat Actor | Attack Vector |
| --- | --- | --- | --- | --- |
| T-001 | Execute unauthorized firmware | AS-001, AS-007 | Remote / Local | Firmware injection via OTA |
| T-002 | Bypass signature verification | AS-007 | Local (physical) | Fault injection |
| T-003 | Extract root key | AS-004 | Advanced (physical) | Side-channel, probing |
| T-004 | Steal signing key from backend | AS-006 | Remote | Backend compromise |
| T-005 | Rollback to vulnerable firmware | AS-010 | Remote / Local | Version counter manipulation |
| T-006 | Clone device identity | AS-003, AS-005 | Local (physical) | Memory readout, key extraction |
| T-007 | Interrupt OTA update | AS-008 | Remote | Network disruption |
| T-008 | Corrupt firmware in transit | AS-001 | Remote (MITM) | Network tampering |
| T-009 | Exploit image header parser | AS-001 | Remote | Malformed image |
| T-010 | Bypass version check | AS-010 | Local (physical) | Glitch injection |

---

## 5. Impact Rating

### 5.1 Impact Categories (ISO 21434)

| Category | Symbol | Description |
| --- | --- | --- |
| Safety | S | Potential for physical harm to persons |
| Financial | F | Direct or indirect financial loss |
| Operational | O | Degradation or loss of vehicle function |
| Privacy | P | Exposure of personal or sensitive data |

### 5.2 Impact Rating Scale

| Rating | Value | Description |
| --- | --- | --- |
| Severe | 4 | Life-threatening, total financial loss, complete function loss, mass data breach |
| Major | 3 | Serious injury possible, major financial loss, significant degradation, large data loss |
| Moderate | 2 | Minor injury possible, moderate financial loss, noticeable degradation, limited data exposure |
| Negligible | 1 | No injury, negligible financial loss, minor inconvenience, no data exposure |

### 5.3 Impact Rating Matrix

| Threat ID | S | F | O | P | Max Impact |
| --- | --- | --- | --- | --- | --- |
| T-001 | 4 | 3 | 4 | 1 | 4 (Severe) |
| T-002 | 4 | 3 | 4 | 1 | 4 (Severe) |
| T-003 | 4 | 4 | 4 | 1 | 4 (Severe) |
| T-004 | 4 | 4 | 4 | 1 | 4 (Severe) |
| T-005 | 3 | 2 | 3 | 1 | 3 (Major) |
| T-006 | 1 | 3 | 3 | 2 | 3 (Major) |
| T-007 | 3 | 3 | 3 | 1 | 3 (Major) |
| T-008 | 4 | 3 | 4 | 1 | 4 (Severe) |
| T-009 | 3 | 2 | 3 | 1 | 3 (Major) |
| T-010 | 3 | 2 | 3 | 1 | 3 (Major) |

---

## 6. Attack Path Analysis

Attack paths are analyzed using the SBOP attack tree (→ `Attack_Tree.md`). Each leaf node in the attack tree represents a concrete attack vector that must be assessed for feasibility.

### 6.1 Attack Path Mapping

| Attack Tree Node | Threat Scenario | Target Asset | Attack Vector Category |
| --- | --- | --- | --- |
| A1.1.1 (Skip verification) | T-002 | AS-007 | Fault Injection |
| A1.1.2 (Parsing bug) | T-009 | AS-001 | Software Exploit |
| A1.2.1 (Break crypto) | T-001 | AS-004 | Cryptanalysis |
| A1.2.2 (Steal signing key) | T-004 | AS-006 | Network Attack |
| A1.3.1 (Tamper during transfer) | T-008 | AS-001 | Network MITM |
| A1.3.2 (Backend compromise) | T-004 | AS-006 | Network Attack |
| A2.1 (Modify after signing) | T-001 | AS-001 | Software |
| A2.2 (Hash verification bug) | T-001 | AS-001 | Software Exploit |
| A3.1 (Install old firmware) | T-005 | AS-010 | Logical Attack |
| A3.2 (Bypass version check) | T-010 | AS-010 | Fault Injection |
| A4.1 (Extract keys) | T-003 | AS-004, AS-005 | Physical Attack |
| A4.2 (Copy identity) | T-006 | AS-003 | Physical Attack |
| A5.1 (Interrupt OTA) | T-007 | AS-008 | Network Attack |
| A5.2 (Corrupt storage) | T-007 | AS-008 | Physical/Logical |

---

## 7. Attack Feasibility Rating

### 7.1 Feasibility Factors (ISO 21434)

| Factor | Description |
| --- | --- |
| Elapsed Time | Time required to identify and exploit vulnerability |
| Expertise | Required knowledge of the item and attack type |
| Knowledge of Item | Required knowledge of SBOP internals |
| Window of Opportunity | When and for how long access is possible |
| Equipment | Required tools and hardware |

### 7.2 Feasibility Scale (CVSS 3.1 aligned)

| Rating | Value | Description |
| --- | --- | --- |
| High | 3 | Attack is practical with common skills/tools |
| Medium | 2 | Attack requires specific expertise or equipment |
| Low | 1 | Attack requires advanced expertise and specialized equipment |
| Very Low | 0 | Theoretically possible but impractical |

### 7.3 Attack Feasibility Ratings

| Attack Node | Time | Expertise | Knowledge | Window | Equipment | Feasibility |
| --- | --- | --- | --- | --- | --- | --- |
| A1.1.1 | Medium | Medium | Low | Unrestricted | Medium | Medium |
| A1.1.2 | Medium | High | Medium | Unrestricted | Low | Low |
| A1.2.1 | High | Very High | Low | Unrestricted | High | Very Low |
| A1.2.2 | Medium | High | Medium | Unrestricted | Medium | Low |
| A1.3.1 | Low | Medium | Medium | Restricted | Medium | Medium |
| A1.3.2 | High | High | Medium | Unrestricted | High | Low |
| A2.1 | Medium | High | Medium | Unrestricted | Low | Low |
| A2.2 | Medium | High | High | Unrestricted | Low | Very Low |
| A3.1 | Low | Low | Low | Unrestricted | Low | High |
| A3.2 | Medium | Medium | Medium | Unrestricted | Medium | Medium |
| A4.1 | High | High | Medium | Restricted | High | Low |
| A4.2 | Medium | Medium | Medium | Restricted | Medium | Medium |
| A5.1 | Low | Low | Low | Restricted | Low | High |
| A5.2 | Medium | Medium | Medium | Restricted | Medium | Medium |

---

## 8. Risk Determination

### 8.1 Risk Formula

```
Risk Value = Impact × Feasibility
```

Where:
- **Impact** = max(S, F, O, P) from impact rating (Scale: 1-4)
- **Feasibility** = attack feasibility rating (Scale: 0-3)

### 8.2 Risk Matrix

| Impact ↓ / Feasibility → | Very Low (0) | Low (1) | Medium (2) | High (3) |
| --- | --- | --- | --- | --- |
| Severe (4) | 0 - Low | 4 - Medium | 8 - High | 12 - Critical |
| Major (3) | 0 - Low | 3 - Low | 6 - Medium | 9 - High |
| Moderate (2) | 0 - Low | 2 - Low | 4 - Low | 6 - Medium |
| Negligible (1) | 0 - Low | 1 - Low | 2 - Low | 3 - Low |

### 8.3 Risk Levels

| Risk Value | Level | Required Action |
| --- | --- | --- |
| 9-12 | Critical | Must mitigate with strong controls; mandatory verification |
| 6-8 | High | Must mitigate or formally justify acceptance |
| 3-5 | Medium | Monitor and reduce where practical |
| 0-2 | Low | Acceptable; periodic review |

### 8.4 SBOP Risk Assessment

| Attack Node | Max Impact | Feasibility | Risk Value | Risk Level |
| --- | --- | --- | --- | --- |
| A1.1.1 (Skip verification) | 4 | 2 (Medium) | 8 | **High** |
| A1.1.2 (Parsing bug) | 3 | 1 (Low) | 3 | Medium |
| A1.2.1 (Break crypto) | 4 | 0 (Very Low) | 0 | Low |
| A1.2.2 (Steal signing key) | 4 | 1 (Low) | 4 | Medium |
| A1.3.1 (Tamper during transfer) | 4 | 2 (Medium) | 8 | **High** |
| A1.3.2 (Backend compromise) | 4 | 1 (Low) | 4 | Medium |
| A2.1 (Modify after signing) | 4 | 1 (Low) | 4 | Medium |
| A2.2 (Hash verification bug) | 3 | 0 (Very Low) | 0 | Low |
| A3.1 (Install old firmware) | 3 | 3 (High) | 9 | **Critical** |
| A3.2 (Bypass version check) | 3 | 2 (Medium) | 6 | High |
| A4.1 (Key extraction) | 4 | 1 (Low) | 4 | Medium |
| A4.2 (Copy identity) | 3 | 2 (Medium) | 6 | High |
| A5.1 (Interrupt OTA) | 3 | 3 (High) | 9 | **Critical** |
| A5.2 (Corrupt storage) | 3 | 2 (Medium) | 6 | High |

---

## 9. Risk Treatment

### 9.1 Treatment Options (ISO 21434)

| Option | Description | Applicable When |
| --- | --- | --- |
| Mitigate | Implement security control to reduce risk | Feasible control exists |
| Accept | Formally accept the residual risk | Risk is low or mitigation impractical |
| Transfer | Share risk with another party (e.g., supplier) | Risk outside direct control |
| Avoid | Remove the source of the risk entirely | Function can be eliminated |

### 9.2 Treatment Decisions

| Attack Node | Risk Level | Treatment | Justification | Control Reference |
| --- | --- | --- | --- | --- |
| A1.1.1 | High | **Mitigate** | Fault injection into verification is a practical attack | Redundant checks, state machine enforcement, → `Fault_Injection_Test.md (FI-001)` |
| A1.1.2 | Medium | **Mitigate** | Parsing bugs are common in C/C++ | Fuzzing, input validation, → `Fuzzing_Strategy.md` |
| A1.2.1 | Low | **Accept** | Breaking Ed25519 computationally infeasible | Use of standardized crypto, → `Crypto_Algorithms.md` |
| A1.2.2 | Medium | **Mitigate** | Signing key is high-value target | HSM protection, key ceremony, → `Key_Management.md` |
| A1.3.1 | High | **Mitigate** | MITM on firmware transfer is practical | Signature verification on device, TLS, → `OTA_Interface.md` |
| A1.3.2 | Medium | **Mitigate** | Backend is trusted but must be hardened | Access control, audit logging, → `Key_Management.md` |
| A2.1 | Medium | **Mitigate** | Integrity check exists | SHA-256 hash verification, → `Crypto_Algorithms.md` |
| A2.2 | Low | **Accept** | Hash algorithm is well-reviewed | Standard SHA-256, no custom logic |
| A3.1 | **Critical** | **Mitigate** | Rollback is trivial without version enforcement | Monotonic version counter, → `REQ-FR-BOOT-004` |
| A3.2 | High | **Mitigate** | Fault injection on version check | Redundant version check, → `Fault_Injection_Test.md (FI-004)` |
| A4.1 | Medium | **Mitigate** | Key extraction via physical attack | Key isolation, secure element, → `Physical_Tamper_Resistance.md` |
| A4.2 | High | **Mitigate** | Identity cloning enables fleet attacks | Identity binding, backend validation, → `Anti_Cloning.md` |
| A5.1 | **Critical** | **Mitigate** | OTA interruption is trivially achievable | Dual-image model, recovery, → `OTA_Recovery.md` |
| A5.2 | High | **Mitigate** | Storage corruption possible | Redundant metadata, → `OTA_Failure_Model.md` |

---

## 10. TARA Evidence Package

### 10.1 Required Evidence

| Evidence ID | Description | TARA Step | Source Document |
| --- | --- | --- | --- |
| TARA-EV-001 | Asset inventory with security properties | Step 1 | This document, Section 3 |
| TARA-EV-002 | Threat scenario catalog | Step 2 | This document, Section 4 |
| TARA-EV-003 | Impact ratings per damage scenario dimension | Step 3 | This document, Section 5 |
| TARA-EV-004 | Attack path descriptions | Step 4 | This document, Section 6; `Attack_Tree.md` |
| TARA-EV-005 | Feasibility assessment per attack path | Step 5 | This document, Section 7 |
| TARA-EV-006 | Risk matrix | Step 6 | This document, Section 8 |
| TARA-EV-007 | Risk treatment decisions and justification | Step 7 | This document, Section 9; `Risk_Treatment.md` |
| TARA-EV-008 | Residual risk statement | Post-treatment | `Residual_Risk.md` |

### 10.2 Audit Checklist

* [ ] Asset inventory is complete and covers all security properties (C, I, A)
* [ ] All threat scenarios are mapped to attack paths
* [ ] Impact ratings are justified for each damage dimension
* [ ] Attack feasibility ratings are documented with rationale
* [ ] Risk formula is applied consistently
* [ ] All Critical/High risks have defined treatment
* [ ] All treatment decisions have justification
* [ ] Residual risks are documented and accepted

---

## 11. Relationship to Internal Risk Model

This TARA methodology uses ISO 21434-compliant 1-4 Impact × 0-3 Feasibility (0-12 scale). A parallel internal risk model (→ `Risk_Quantification.md`) uses a 1-5 Likelihood × 1-5 Impact scale (1-25) for finer-grained engineering decisions.

The TARA ratings are the compliance-auditable values. Internal ratings may exceed TARA ratings to drive additional engineering review, but never fall below them.

---

## 13. TARA Maintenance

The TARA shall be reviewed and updated:

* When the system architecture changes significantly
* When a new vulnerability class affecting SBOP is discovered
* When new attack techniques become practical (e.g., quantum computing advances)
* At minimum, annually as part of the cybersecurity assessment cycle

Changes to the TARA must be traceable and justify any modifications to risk ratings or treatment decisions.
