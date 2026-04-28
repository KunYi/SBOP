# Residual Risk

**Document ID:** SEC-RISK-RES-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document identifies risks that cannot be fully mitigated by SBOP's security controls and must be formally accepted. Each residual risk includes the rationale for acceptance and any monitoring controls.

---

## 2. Residual Risk Register

### 2.1 Physical Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-PHY-001 | Advanced decapsulation and microprobing (FIB) | Accepted for SL ≤ 3 | Requires semiconductor lab resources (nation-state level). Mitigated at SL 4 via secure element with active shield. | Tamper event logging |
| RRES-PHY-002 | Laser fault injection | Accepted for SL ≤ 2 | Requires specialized and expensive equipment. Partially mitigated by redundant checks at SL 2+. Monitored at SL 3+. | Glitch counter monitoring |
| RRES-PHY-003 | Physical destruction of device | Accepted (all SL) | Cannot be prevented by software. Mitigated by key zeroization on tamper detection, limiting data exposure. | — |
| RRES-PHY-004 | Advanced EM fault injection | Accepted for SL ≤ 3 | Requires custom EM probe setup and expertise. Mitigated at SL 4 via active shielding. | EM TVLA testing |

### 2.2 Side-Channel Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-SCH-001 | High-trace-count DPA (> 1M traces) | Accepted for SL ≤ 2 | Requires extended physical access and signal processing expertise. Mitigated by masking at SL 3+. | TVLA periodic re-test |
| RRES-SCH-002 | Professional EM analysis | Accepted for SL ≤ 2 | Requires specialized EM lab. Mitigated by shielding at SL 3+. | EM TVLA periodic re-test |
| RRES-SCH-003 | Remote timing attacks on network-facing OTA | Accepted (all SL) | Network jitter adds significant noise to timing measurements. Constant-time crypto limits leakage. Monitoring of OTA timing anomalies. | Network telemetry |

### 2.3 Supply Chain Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-SC-001 | Insider threat with signing key access | Accepted (monitored) | Multi-party quorum and HSM enforcement reduce but cannot eliminate collusion risk. All signing operations logged. | Signing audit review |
| RRES-SC-002 | Zero-day vulnerability in crypto library dependency | Accepted (monitored) | SBOP uses standard, reviewed libraries. CVE scanning at build time detects known vulns. Unknown vulns are residual. | CVE monitoring; update readiness |
| RRES-SC-003 | Counterfeit MCU / secure element | Accepted (monitored) | Detectable if device identity verification fails. Undetectable if counterfeit perfectly replicates genuine hardware. | Backend device attestation |

### 2.4 Debug Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-DBG-001 | Debug auth credential compromise at provisioning station | Accepted (monitored) | Station compromise during provisioning window is a narrow but real threat. KD_Debug is device-specific; compromise is per-device. | Station security audit |
| RRES-DBG-002 | Silicon-level debug backdoor (vendor test interface) | Accepted (all SL) | SBOP cannot control silicon vendor debug features. Relies on vendor security claims. | Vendor security assessment |

### 2.5 Manufacturing Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-MFR-001 | Unauthorized overproduction at rogue factory | Accepted (monitored) | Backend registration count detects batch-level overproduction. Individual device overproduction may go undetected if registration is bypassed. | Batch count audit |
| RRES-MFR-002 | Provisioning data leak from compromised factory network | Accepted (monitored) | Factory network isolation and encryption limit exposure. Data leak may expose per-device records but not keys (wrapped). | Factory audit |
| RRES-MFR-003 | Non-provisioned devices entering field | Accepted (monitored) | Devices failing provisioning are rejected at factory line. Process failures could allow escape. Backend blocks unregistered devices. | Backend enrollment check |

### 2.6 Aviation-Specific Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-AV-001 | DO-178C Level A: tool qualification gaps | Accepted (pre-implementation) | Tool qualification is an implementation-phase activity. Accepted at Tier-1 level with commitment to address during implementation. | — |
| RRES-AV-002 | CAST-32A multi-core interference | Accepted (pre-implementation) | Analysis deferred to platform selection. SBOP runs on locked single core during boot; interference from other cores is a platform concern. | Platform analysis |

### 2.7 Space-Specific Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-SP-001 | Single Event Upset (SEU) in boot state machine during verification | Accepted (mitigated) | TMR/EDAC is a hardware platform concern. SBOP specification requires redundant checks and fail-safe on state integrity failure. | SEU monitoring |
| RRES-SP-002 | Total ionizing dose (TID) degrading flash storage over mission life | Accepted (all SL) | Flash endurance is a hardware concern. SBOP's dual-image model allows failing-over from degraded slots. | Slot error rate monitoring |

### 2.8 Medical-Specific Domain

| Risk ID | Risk Description | Residual Level | Justification | Monitoring |
| --- | --- | --- | --- | --- |
| RRES-MED-001 | IEC 62304 Class C: insufficient segregation evidence | Accepted (pre-implementation) | Zone 1/Zone 2 memory segregation is specified at Tier-1. Verification evidence requires implementation. | Post-implementation verification |
| RRES-MED-002 | Off-label use of SOUP components | Accepted (monitored) | Crypto library validated for general use; medical-specific validation is the integrator's responsibility. | SOUP anomaly monitoring |

---

## 3. Risk Acceptance Authority

All residual risks require formal acceptance by:

| Role | Responsibility |
| --- | --- |
| Security Architect | Technical assessment of residual risk level |
| Product Manager | Business decision to accept risk |
| Compliance Officer | Confirmation that residual risk is compatible with target standards |
| Customer / Integrator | Acceptance of residual risk for their deployment context |

---

## 4. Residual Risk Review Cycle

| Trigger | Action |
| --- | --- |
| New vulnerability class affecting SBOP | Re-assess all related residual risks |
| Annual security review | Full residual risk register review |
| Before each certification audit | Validate all accepted risks are still acceptable |
| Major architecture change | Re-assess all residual risks |
