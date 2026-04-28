# Non-Functional Requirements

**Document ID:** REQ-NFR-SYS-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## REQ-NFR-SYS-001: Platform Agnosticism

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The SBOP specification shall be hardware platform agnostic. It shall define interfaces, state machines, data structures, and algorithms without depending on any specific MCU architecture, vendor SDK, or hardware feature. Reference implementations may target specific platforms, but the specification must be implementable on any platform meeting the minimum requirements.

**Rationale:**
Enables reuse across product lines, MCU vendors, and deployment domains. Prevents vendor lock-in.

**Verification:** REVIEW
**Design:** `Reference_Architecture.md` §8 (minimum requirements, recommended platforms)
**Metric:** Specification must reference no vendor-specific registers, SDK APIs, or hardware features.

---

## REQ-NFR-SYS-002: Scalable Backend Integration

**Priority:** P1 (Required)
**Status:** Approved

**Description:**
The system shall support scalable backend integration capable of managing fleets from 10 to 10,000,000+ devices. Backend interfaces shall use standard protocols (HTTPS, TLS 1.3, REST or gRPC) and shall not impose per-device state that prevents horizontal scaling.

**Rationale:**
SBOP targets both small-scale industrial deployments and large-scale automotive/IoT fleets. Backend architecture must scale without redesign.

**Verification:** REVIEW, TEST (load test at scale)
**Design:** `Reference_Architecture.md` §2 (backend components)
**Metric:** Backend must support ≥ 100,000 concurrent OTA sessions.

---

## REQ-NFR-SYS-003: Deterministic and Reproducible Build

**Priority:** P1 (Required)
**Status:** Approved

**Description:**
The system shall support deterministic, bit-for-bit reproducible builds. Identical source code and toolchain shall produce identical binaries. Build inputs shall be pinned (compiler version, libraries, flags, environment). An SBOM (SPDX or CycloneDX) shall be generated at build time.

**Rationale:**
Enables independent verification that the signed binary matches the source code. Required for supply chain security and compliance (ISO 21434 §10).

**Verification:** TEST (reproducible build verification)
**Design:** `Supply_Chain_Security.md` §3.1, `Reference_Architecture.md` §4
**Related Requirements:** REQ-SR-SC-001

---

## REQ-NFR-SYS-004: Boot Latency Budget

**Priority:** P1 (Required)
**Status:** Approved

**Description:**
The SBOP boot process shall complete within a defined time budget. Target budgets by deployment:

| Context | Boot Time Budget |
| --- | --- |
| Automotive (ASIL B+) | 100 ms from RESET to EXECUTE |
| Industrial (SIL 2) | 500 ms |
| Medical (Class C) | 250 ms |
| Space (Cat B) | 5 seconds (SEU scrubbing included) |
| General embedded | 1 second |

Timing is measured from RESET de-assertion to application entry point (excluding application initialization).

**Rationale:**
Safety-critical systems have fault-tolerant time intervals (FTTI). Boot must complete within the FTTI. Automotive ECUs typically require < 100 ms boot.

**Verification:** TEST (timing measurement with oscilloscope)
**Design:** `Boot_Flow_Pseudocode.md` §11 (timing budgets)
**Metric:** Measured timing ≤ budget on target platform.

---

## REQ-NFR-SYS-005: Resource-Constrained Operation

**Priority:** P1 (Required)
**Status:** Approved

**Description:**
The system shall operate within constrained resource environments as defined by the minimum platform requirements:

| Resource | Minimum | Recommended |
| --- | --- | --- |
| Flash (bootloader) | 64 KB | 128 KB |
| Flash (per firmware slot) | Application-dependent | — |
| RAM | 32 KB | 64 KB |
| CPU | Cortex-M0+ / RISC-V RV32IMC | Cortex-M33 with TrustZone |
| Secure storage | 4 KB OTP or write-protected flash | Secure element |

**Rationale:**
SBOP targets cost-sensitive embedded devices. It must not require premium hardware features for basic security.

**Verification:** TEST (verify on minimum platform)
**Design:** `Reference_Architecture.md` §8.1
**Metric:** Bootloader flash ≤ 64 KB, RAM usage ≤ 32 KB.

---

## REQ-NFR-SYS-006: Multi-Standard Compliance

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The SBOP specification shall be designed to support compliance with the following standards:
- ISO/SAE 21434 (Automotive cybersecurity)
- IEC 62443-4-1/4-2 (Industrial security)
- ECSS-E-ST-40 / Q-ST-80 (Space software)
- IEC 62304 / ISO 14971 (Medical device software)
- DO-178C / DO-326A (Aviation software)

Compliance evidence shall be traceable from requirement → design → implementation → test → evidence.

**Rationale:**
SBOP is a cross-domain specification. Standards compliance is required for product certification in each domain.

**Verification:** AUDIT
**Design:** `Compliance_Overview.md`, `Audit_Trace.md`
**Related Requirements:** All REQ-SR-SC, REQ-SR-PHY, REQ-SR-SCH, REQ-SR-DBG, REQ-SR-MFR

---

## REQ-NFR-SYS-007: Security by Default

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
All security-critical features shall be enabled by default. No security feature shall require configuration to be active. Examples: signature verification always on, anti-rollback always enforced, debug port always locked after provisioning. Any deviation requires explicit justification and acceptance in the residual risk register.

**Rationale:**
"Optional security" is effectively "no security" — operators will disable it for convenience. Fail-closed, secure-by-default is a core principle.

**Verification:** REVIEW, TEST
**Design:** All security architecture documents
**Related Requirements:** REQ-SR-DBG-001, REQ-SR-BOOT-001
