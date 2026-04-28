# Verification Coverage Model

**Document ID:** VER-COV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the coverage model for SBOP verification completeness. Specifies coverage types, targets, measurement methods, and gap analysis procedures across functional, structural, requirement, fault, and security dimensions.

---

## 2. Coverage Dimensions

### 2.1 Functional Coverage

**Target:** All system flows exercised with valid and invalid inputs.

| Coverage Item | Method | Target |
| --- | --- | --- |
| Boot flow (normal path) | TEST-BOOT-001 | 100% of phases executed |
| Boot flow (each error path) | TEST-BOOT-002 through TEST-BOOT-005 | Every error code triggered |
| OTA flow (normal path) | TEST-OTA-001 | All phases executed |
| OTA flow (interruption at each phase) | TEST-OTA-002 | Each phase interrupted + recovered |
| Provisioning flow | Factory test | All provisioning steps |
| Debug lifecycle (all 4 phases) | DBG-TEST-001 | Each phase transition tested |
| Tamper response (all levels) | FI-001, Physical pentest | Each response level triggered |

### 2.2 State Coverage

**Target:** 100% state coverage on all state machines.

| State Machine | States | Transitions | Coverage Target |
| --- | --- | --- | --- |
| Boot state machine | 13 states | ~20 transitions | 100% states, 100% transitions |
| OTA state machine | 7 states | ~15 transitions | 100% states, 100% transitions |
| Provisioning state machine | 5 states | ~8 transitions | 100% states, 100% transitions |
| Debug state machine | 6 states | ~10 transitions | 100% states, 100% transitions |
| Slot status (per slot) | 5 states | ~8 transitions | 100% states, 100% transitions |

### 2.3 Requirement Coverage

**Target:** Each requirement (REQ-FR-xxx, REQ-SR-xxx, REQ-NFR-xxx) mapped to ≥ 1 test case.

| Requirement Category | Count | Tests Mapped | Coverage |
| --- | --- | --- | --- |
| REQ-FR-BOOT | 5 | TEST-BOOT-001..005 | 100% |
| REQ-FR-OTA | 5 | TEST-OTA-001..005 | 100% |
| REQ-SR-BOOT | 2 | TEST-BOOT-002, TEST-BOOT-003 | 100% |
| REQ-SR-OTA | 3 | TEST-OTA-001..003 | 100% |
| REQ-SR-KEY | 2 | Analysis, Signing audit | 100% |
| REQ-SR-ID | 2 | TEST-ID-001, TEST-ID-002 | 100% |
| REQ-SR-SC | 2 | SC-AUDIT-001..004 | 100% |
| REQ-SR-PHY | 2 | FI-001, Physical pentest | 100% |
| REQ-SR-SCH | 1 | TIM-001, TIM-002, TVLA | 100% |
| REQ-SR-DBG | 2 | DBG-TEST-001, DBG-TEST-002 | 100% |
| REQ-SR-MFR | 2 | Factory audit | 100% |
| REQ-NFR-SYSTEM | 4 | Performance, security audit tests | 100% |

### 2.4 Fault Coverage

**Target:** All defined fault scenarios tested.

| Fault Category | Count | Testing Method |
| --- | --- | --- |
| Cryptographic failures | 8 | TEST-BOOT-002, TEST-OTA-003 |
| Storage failures | 6 | TEST-BOOT-003, TEST-OTA-003 |
| Network failures | 5 | TEST-OTA-002 |
| Power loss during operation | 4 | FI-001, TEST-OTA-002 |
| Hardware faults | 4 | FI-001 |
| Fault injection (glitch) | 6 | FI-001 through FI-004 |
| Tamper events | 5 | Physical pentest |

### 2.5 Attack Coverage

**Target:** Every attack tree leaf node has a corresponding test.

| Attack Root | Leaf Nodes | Tests Mapped | Coverage |
| --- | --- | --- | --- |
| A1: Unauthorized Firmware | 6 | 6 | 100% |
| A2: Firmware Integrity | 2 | 2 | 100% |
| A3: Rollback | 2 | 2 | 100% |
| A4: Device Cloning | 2 | 2 | 100% |
| A5: OTA Exploitation | 2 | 2 | 100% |
| A6: Supply Chain | 8 | 5 (3 process-only) | 100% |
| A7: Physical Tampering | 8 | 6 (2 SL4-only) | 100% |
| A8: Side-Channel | 5 | 5 | 100% |
| A9: Debug Exploitation | 6 | 6 | 100% |
| A10: Manufacturing | 6 | 4 (2 audit-only) | 100% |
| A11: Zone Boundary | 6 | 6 | 100% |
| **Total** | **53** | **46 test + 7 process** | **100%** |

Note: "Process-only" leaves are verified by audit/procedure, not automated testing.

### 2.6 Standards Coverage

| Standard | Clauses Mapped | Evidence Items | Coverage |
| --- | --- | --- | --- |
| ISO 21434 | Clauses 5-15 | 15 evidence items | 100% of applicable |
| IEC 62443 | FR 1-7, 50+ CRs | Per-CR evidence | 100% mapped |
| ECSS | E-ST-40, Q-ST-80, E-ST-70-41 | 20+ trace items | 100% mapped |
| IEC 62304 | Clauses 4-8 | Per-clause evidence | 100% mapped |
| DO-178C | DAL A-E objectives | 71 objectives Level A | 100% mapped |

---

## 3. Coverage Measurement

### 3.1 Structural Coverage (Code)

For C implementations (post-specification phase):

| Coverage Type | DAL A (DO-178C) | ASIL D (ISO 26262) | SIL 4 (IEC 61508) |
| --- | --- | --- | --- |
| Statement coverage | 100% | 100% | 100% |
| Branch/Decision coverage | 100% | 100% | 100% |
| MC/DC | Required | Recommended | Recommended |
| Function coverage | 100% | 100% | 100% |
| Function call coverage | 100% | 100% | 100% |

### 3.2 Requirements-Based Coverage

Each requirement must have:
1. ≥ 1 normal-path test (verifies requirement is met)
2. ≥ 1 error-path test (verifies safe handling of violations)
3. ≥ 1 boundary test (verifies edge conditions)

### 3.3 Coverage Metrics Calculation

```
Functional Coverage = (exercised_flows / total_flows) × 100%
State Coverage = (tested_transitions / total_transitions) × 100%
Requirement Coverage = (tested_requirements / total_requirements) × 100%
Fault Coverage = (tested_faults / defined_faults) × 100%
Attack Coverage = (tested_leaf_nodes / total_leaf_nodes) × 100%
```

---

## 4. Gap Analysis Procedure

### 4.1 Gap Identification

| Step | Description |
| --- | --- |
| 1 | Run coverage measurement on latest test results |
| 2 | Identify any requirement with < 1 test |
| 3 | Identify any state/transition not visited |
| 4 | Identify any attack leaf node without test |
| 5 | Identify any error code not triggered |
| 6 | Document gaps in coverage gap register |

### 4.2 Gap Resolution

| Gap Priority | Resolution Timeline |
| --- | --- |
| Critical (safety/security requirement uncovered) | Must resolve before release |
| High (security requirement partially covered) | Resolve within 1 sprint |
| Medium (non-critical requirement uncovered) | Resolve within release cycle |
| Low (edge case, unlikely in practice) | Document and accept |

---

## 5. Coverage Tools

| Tool | Coverage Type | Notes |
| --- | --- | --- |
| gcov / lcov | Statement, branch | C/C++ code coverage |
| kcov | Statement, branch | Language-agnostic |
| TLA+ TLC | State machine coverage | Model checking coverage |
| CBMC | Bounded model checking | Formal coverage of C code |
| Custom coverage mapper | Requirement → test trace | See Traceability_Matrix.md |

---

## 6. Coverage Reports

### 6.1 Report Template

```
Coverage Report
===============
Date: YYYY-MM-DD
Release: X.Y.Z

Functional Coverage: XX% (Y/Z flows)
State Coverage: XX% (Y/Z transitions)
Requirement Coverage: XX% (Y/Z requirements)
Fault Coverage: XX% (Y/Z fault scenarios)
Attack Coverage: XX% (Y/Z leaf nodes)

Gaps:
  - GAP-001: Requirement REQ-xxx uncovered → planned test TEST-xxx
  - GAP-002: State transition X→Y not tested → planned in FI-00x

Uncovered Justifications:
  - REQ-xxx not testable in simulation (requires physical access) → covered by audit
```

### 6.2 Review Cadence

| Event | Coverage Review |
| --- | --- |
| Each test execution | Update coverage metrics |
| Each release | Full coverage report |
| Pre-certification audit | Coverage evidence package |
| Annual review | Coverage model adequacy review |

---

## 7. References

| Document | Reference |
| --- | --- |
| Test Cases | `Test_Cases.md` |
| Traceability Matrix | `Traceability_Matrix.md` |
| Fault Injection Test | `Fault_Injection_Test.md` |
| Red Team Test Plan | `Red_Team_Test_Plan.md` |
| Fuzzing Strategy | `Fuzzing_Strategy.md` |
| Attack Tree | `../04_Security/Attack_Tree.md` |
| Requirements | `../01_Requirements/` |
