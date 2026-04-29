# Risk Quantification

**Document ID:** SEC-RQ-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines risk evaluation model for threats.

---

## 2. Risk Formula

Risk = Likelihood × Impact

---

## 3. Likelihood Levels

1 - Very Low (theoretical)
2 - Low (difficult)
3 - Medium (feasible)
4 - High (practical)
5 - Very High (trivial)

---

## 4. Impact Levels

1 - Negligible
2 - Minor
3 - Moderate
4 - Major
5 - Critical

---

## 5. Risk Levels

1–5 → Low
6–10 → Medium
11–15 → High
16–25 → Critical

---

## 6. Risk Table

| Attack ID | Description | Likelihood | Impact | Risk |
| --------- | ----------- | ---------- | ------ | ---- |

A1.1.1 | Skip verification (fault injection) | 4 | 5 | 20 (Critical)
A1.1.2 | Exploit parsing bug | 3 | 5 | 15 (High)
A1.2.1 | Break crypto | 1 | 5 | 5 (Low)
A1.2.2 | Steal signing key | 3 | 5 | 15 (High)
A1.3.1 | Tamper during firmware transfer | 3 | 5 | 15 (High)
A1.3.2 | Backend compromise | 2 | 5 | 10 (High)
A2.1 | Modify firmware after signing | 2 | 5 | 10 (High)
A2.2 | Hash verification bug | 1 | 5 | 5 (Low)
A3.1 | Install old firmware (rollback) | 4 | 4 | 16 (Critical)
A3.2 | Bypass version check | 3 | 4 | 12 (High)
A4.1 | Key extraction | 3 | 5 | 15 (High)
A4.2 | Copy identity data | 3 | 4 | 12 (High)
A5.1 | Interrupt OTA | 5 | 3 | 15 (High)
A5.2 | Corrupt storage | 4 | 3 | 12 (High)
A6.1.1 | Malicious commit (insider) | 2 | 5 | 10 (High)
A6.1.2 | Compromised developer credentials | 2 | 5 | 10 (High)
A6.2.1 | Build server compromise | 3 | 5 | 15 (High)
A6.2.2 | Malicious build tool/dependency | 3 | 5 | 15 (High)
A6.3.1 | Steal signing key from HSM | 1 | 5 | 5 (Low)
A6.3.2 | Sign unauthorized via insider | 2 | 5 | 10 (High)
A6.4.1 | Replace firmware on CDN | 3 | 4 | 12 (High)
A6.4.2 | MITM on firmware download | 3 | 4 | 12 (High)
A7.1.1 | Bus sniffing | 3 | 4 | 12 (High)
A7.1.2 | Flash readout (chip-off) | 3 | 4 | 12 (High)
A7.2.1 | Voltage glitching | 2 | 5 | 10 (High)
A7.2.2 | Clock glitching | 2 | 5 | 10 (High)
A7.2.3 | EM fault injection | 2 | 5 | 10 (High)
A7.3.1 | Open enclosure | 3 | 4 | 12 (High)
A7.3.2 | Bypass tamper switch | 2 | 4 | 8 (High)
A8.1.1 | Timing oracle (signature verify) | 3 | 4 | 12 (High)
A8.1.2 | Timing oracle (hash compare) | 3 | 3 | 9 (High)
A8.2.1 | SPA on Ed25519 verification | 2 | 4 | 8 (High)
A8.2.2 | DPA on key derivation | 2 | 4 | 8 (High)
A8.3.1 | EM leakage from crypto ops | 2 | 4 | 8 (High)
A9.1.1 | Debug left open after provisioning | 2 | 5 | 10 (High)
A9.1.2 | Debug auth bypass | 1 | 5 | 5 (Low)
A9.2.1 | Steal debug auth credentials | 2 | 4 | 8 (High)
A9.2.2 | Brute-force debug auth challenge | 2 | 5 | 10 (High)
A9.3.1 | Breakpoint injection during boot | 2 | 5 | 10 (High)
A10.1.1 | Overproduction | 3 | 3 | 9 (High)
A10.2.1 | Clone UID at factory | 2 | 5 | 10 (High)
A10.2.2 | Copy identity between devices | 2 | 4 | 8 (High)
A10.3.1 | Intercept key injection | 2 | 5 | 10 (High)
A10.4.1 | Ship with test/debug enabled | 2 | 4 | 8 (High)
A10.4.2 | Unauthorized firmware via test mode | 2 | 4 | 8 (High)
A11.1.1 | Application reads boot memory | 3 | 5 | 15 (High)
A11.1.2 | OTA writes to active slot | 2 | 5 | 10 (High)
A11.2.1 | Bypass TLS mutual auth | 3 | 5 | 15 (High)
A11.2.2 | Replay captured OTA traffic | 3 | 4 | 12 (High)
A11.3.1 | Unauthorized station connects to backend | 2 | 5 | 10 (High)
A11.3.2 | Station malware exfiltrates keys | 2 | 5 | 10 (High)

---

## 7. Risk Treatment Rules

Critical:
→ Must mitigate with strong controls

High:
→ Must mitigate or justify

Medium:
→ Monitor and reduce

Low:
→ Acceptable

---

## 8. Design Impact

Risk must drive:

* Requirement updates
* Design reinforcement
* Additional testing

---

## 9. Relationship to TARA Methodology

This document uses a simplified 1-5 Likelihood × 1-5 Impact scale for internal risk communication. The formal ISO 21434 TARA (→ `TARA_Methodology.md`) uses a 1-4 Impact × 0-3 Feasibility scale per ISO 21434 §9.5.

Mapping between the two systems:

| Risk_Quantification | TARA_Methodology | Usage |
| --- | --- | --- |
| Likelihood (1-5) | Feasibility (0-3) | Internal: 5-level granularity for engineering decisions |
| Impact (1-5) | Impact S/F/O/P (1-4) | TARA: four-dimensional impact per ISO 21434 |
| Risk = L × I (1-25) | Risk = I × F (0-12) | Different formulas; TARA is the audit-facing metric |

When a risk level differs between the two systems, the TARA assessment takes precedence for compliance reporting. Internal risk ratings may be more conservative to drive additional engineering review.
