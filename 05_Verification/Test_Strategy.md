# SBOP Verification Strategy

**Document ID:** VER-STRAT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the overall verification strategy for SBOP: levels, methods, tools, environments, entry/exit criteria, and traceability to standards requirements.

---

## 2. Verification Goals

| Goal | Description | Measurement |
| --- | --- | --- |
| G-VER-01 | All requirements satisfied | 100% requirement coverage |
| G-VER-02 | All security properties validated | All attack tree leaves tested or audited |
| G-VER-03 | All failure modes verified | Every error code triggered in test |
| G-VER-04 | No undefined behavior | ASAN/UBSAN clean; fuzzing no crashes |
| G-VER-05 | Standards evidence generated | Evidence package per standard |

---

## 3. Verification Levels

### 3.1 Unit Level

| Scope | Method | Tools | Coverage Target |
| --- | --- | --- | --- |
| Crypto functions | Unit tests + test vectors | C test framework, NIST vectors | 100% branch |
| Parsing logic | Unit tests + fuzzing | libFuzzer, AFL++ | 100% branch, 95%+ line |
| State machine transitions | Unit tests | Custom harness | 100% transitions |
| Error handlers | Fault injection (simulated) | Unit test mock injection | All error codes triggered |

### 3.2 Subsystem Level

| Scope | Method | Tools | Coverage Target |
| --- | --- | --- | --- |
| Boot behavior | Integration tests (TEST-BOOT-001..005) | Test framework, HW or emulator | All boot phases |
| OTA flow | Integration tests (TEST-OTA-001..004) | Test framework + mock backend | All OTA phases |
| Identity provisioning | Integration tests (TEST-ID-001..003) | Factory test station | All provisioning steps |
| Crypto subsystem | Integration + TIM tests | Oscilloscope, ct-verif | Constant-time verified |

### 3.3 System Level

| Scope | Method | Tools | Coverage Target |
| --- | --- | --- | --- |
| End-to-end OTA | System tests (boot → OTA → boot) | Full stack: device + backend | Complete update cycle |
| Boot + OTA integration | System tests | Full stack | Slot transitions, rollback |
| Provisioning + registration | System tests | Station + backend | Full provisioning flow |

### 3.4 Security Level

| Scope | Method | Tools | Coverage Target |
| --- | --- | --- | --- |
| Attack simulation | Red team testing | Per Red_Team_Test_Plan.md | All attack leaves |
| Fault injection | HW fault injection | ChipWhisperer, voltage glitcher | Critical check bypass |
| Side-channel | TVLA, timing measurement | Oscilloscope, ct-verif | No leakage |
| Fuzzing | Coverage-guided fuzzing | libFuzzer, AFL++ | All parsers, 0 crashes |

---

## 4. Verification Methods

| Method | When Applied | Evidence Produced |
| --- | --- | --- |
| Functional testing | All requirements (test cases) | Test reports with pass/fail |
| Negative testing | All parsers, state machines | Error handling verification |
| Fuzzing | All input parsers (see Fuzzing_Strategy.md) | Fuzz campaign reports |
| Fault injection | Critical security paths (see Fault_Injection_Test.md) | FI reports |
| Timing measurement | All crypto operations | TIM-001, TIM-002 reports |
| TVLA | Key derivation, signature verification | TVLA test reports |
| Static analysis | All code | Zero-warning report |
| Formal verification | Boot state machine, crypto properties | Model checker output |
| Penetration testing | Complete system (see Red_Team_Test_Plan.md) | Red team report |
| Audit | Procedural controls (supply chain, manufacturing) | Audit reports |

---

## 5. Test Environment

### 5.1 Environments

| Environment | Purpose | Fidelity |
| --- | --- | --- |
| Host simulation | Unit tests, fuzzing, static analysis | Low (no hardware) |
| Emulator (QEMU, Renode) | Integration tests, state machine tests | Medium |
| Reference platform | System tests, performance, timing | High |
| Target hardware | Final verification, fault injection, side-channel | Exact |

### 5.2 Tools

| Tool | Purpose |
| --- | --- |
| Ceedling / Unity | C unit test framework |
| CMock | Mock generation for C |
| libFuzzer / AFL++ | Coverage-guided fuzzing |
| ASAN / UBSAN | Memory safety, undefined behavior |
| Valgrind | Memory leak detection |
| ChipWhisperer | Voltage/clock fault injection |
| J-Link / OpenOCD | Debug access, flash programming |
| Oscilloscope (1 GHz+) | Timing measurement |
| ct-verif / SideTrail | Constant-time verification |
| TLA+ / nuXmv | Model checking |
| gcov / lcov | Code coverage |

---

## 6. Entry and Exit Criteria

### 6.1 Per Test Level

| Level | Entry Criteria | Exit Criteria |
| --- | --- | --- |
| Unit | Code compiles with zero errors | 100% branch coverage, 0 ASAN/UBSAN errors |
| Subsystem | Unit tests pass | All TEST-* test cases pass; all error codes triggered |
| System | Subsystem tests pass | All end-to-end scenarios pass; no regressions |
| Security | System tests pass | All red team tests pass; TVLA pass; fuzzing no new crashes for 24h |

### 6.2 Per Release

| Gate | Criteria |
| --- | --- |
| Alpha | All unit + subsystem tests pass |
| Beta | All system tests pass; security tests pass for SL target |
| RC | All tests pass; fuzzing campaign complete (7 days); coverage targets met |
| Release | All RC criteria + audit sign-off + evidence package complete |

---

## 7. Standards Requirements

| Standard | Verification Requirements | SBOP Coverage |
| --- | --- | --- |
| DO-178C DAL A | 66 objectives; MC/DC on all Level A code | TEST-BOOT-001..005 + formal verification |
| DO-178C DAL C | Statement + decision coverage | TEST-OTA-001..004 |
| ISO 26262 ASIL D | Unit + integration + fault injection | All TEST-* + FI-001..004 |
| IEC 61508 SIL 3 | Static analysis + dynamic testing + formal | All methods at §4 |
| IEC 62304 Class C | Detailed design verification | All test levels |

---

## 8. Regression Strategy

| Trigger | Regression Scope |
| --- | --- |
| Code change to bootloader | Full boot test suite + fault injection |
| Code change to crypto | Full crypto test suite + timing + TVLA |
| Code change to OTA | Full OTA test suite |
| Dependency update | Affected subsystem + fuzzing |
| Platform change | Full test suite on new platform |

---

## 9. References

| Document | Reference |
| --- | --- |
| Test Cases | `Test_Cases.md` |
| Coverage Model | `Coverage_Model.md` |
| Fault Injection Test | `Fault_Injection_Test.md` |
| Fuzzing Strategy | `Fuzzing_Strategy.md` |
| Red Team Test Plan | `Red_Team_Test_Plan.md` |
| Traceability Matrix | `Traceability_Matrix.md` |
