# Security Traceability

**Document ID:** SEC-TRACE-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the mapping from threats to system protections and verification.

---

## 2. Trace Structure

Threat → Attack Scenario → Mitigation → Requirement → Design → Test

---

## 3. Trace Matrix

### T-001: Unauthorized Firmware Execution

Attack Scenario:
Attacker attempts to run malicious firmware

Mitigation:

* Signature verification
* Root of trust enforcement

Requirement:

* REQ-FR-BOOT-001
* REQ-SR-BOOT-001

Design:

* Boot_Overview
* Crypto_Algorithms

Test:

* TEST-BOOT-002

---

### T-002: Firmware Tampering

Attack Scenario:
Firmware modified after signing

Mitigation:

* Hash verification (SHA-256)

Requirement:

* REQ-FR-BOOT-002

Design:

* Image_Format (hash field)
* Crypto_Algorithms

Test:

* TEST-BOOT-003

---

### T-003: Rollback Attack

Attack Scenario:
Old vulnerable firmware is installed

Mitigation:

* Version enforcement

Requirement:

* REQ-FR-BOOT-004
* REQ-SR-OTA-002

Design:

* Boot_State_Detail
* OTA_Recovery

Test:

* TEST-BOOT-004

---

### T-004: Device Cloning

Attack Scenario:
Copy identity to another device

Mitigation:

* Identity binding
* Backend validation

Requirement:

* REQ-SR-ID-001
* REQ-SR-ID-002

Design:

* Identity_Overview
* Anti_Cloning

Test:

* TEST-ID-002

---

### T-005: OTA Tampering

Attack Scenario:
Firmware modified during transfer

Mitigation:

* Signature verification after download

Requirement:

* REQ-FR-OTA-002
* REQ-SR-BOOT-001

Design:

* OTA_State_Machine
* Crypto

Test:

* TEST-OTA-003

---

### T-006: Fault Injection (Skip Verification)

Attack Scenario:
Skip signature verification step

Mitigation:

* State machine enforcement
* Fail-safe design

Requirement:

* REQ-SR-BOOT-001

Design:

* Boot_State_Detail
* Fault_Injection_Test

Test:

* FI-001

---

### T-007: Key Extraction

Attack Scenario:
Attacker attempts to extract keys

Mitigation:

* Key isolation
* No key exposure

Requirement:

* REQ-SR-KEY-001

Design:

* Crypto Implementation Constraints

Test:

* Analysis / Review

---

### T-008: Supply Chain — Build Compromise

Attack Scenario:
Attacker compromises the build environment to inject malicious code.

Mitigation:
* Reproducible builds
* Code signing
* SBOM verification

Requirement:
* REQ-SR-SC-001
* REQ-SR-SC-002

Design:
* Supply_Chain_Security

Test:
* SC-AUDIT-001

---

### T-009: Supply Chain — Signing Key Compromise

Attack Scenario:
Attacker gains access to firmware signing key.

Mitigation:
* HSM protection
* Multi-party signing
* Key ceremony

Requirement:
* REQ-SR-KEY-001

Design:
* Supply_Chain_Security (Section 4)
* Key_Management

Test:
* Analysis / Audit

---

### T-010: Physical Tamper — Glitch Injection

Attack Scenario:
Attacker injects voltage or clock glitch to skip verification.

Mitigation:
* Redundant checks
* Voltage/clock monitors
* Fail-safe on glitch detection

Requirement:
* REQ-SR-PHY-001
* REQ-SR-PHY-002

Design:
* Physical_Tamper_Resistance (Section 7.2)
* Fault_Injection_Test

Test:
* FI-001, FI-004

---

### T-011: Physical Tamper — Enclosure Bypass

Attack Scenario:
Attacker opens device enclosure without triggering tamper detection.

Mitigation:
* Tamper switch
* Active tamper response
* Key zeroization

Requirement:
* REQ-SR-PHY-001
* REQ-SR-PHY-002

Design:
* Physical_Tamper_Resistance (Section 6)

Test:
* Physical penetration test

---

### T-012: Side-Channel — Timing Attack

Attack Scenario:
Attacker measures execution time to extract key information.

Mitigation:
* Constant-time crypto
* Fixed-time comparisons

Requirement:
* REQ-SR-SCH-001

Design:
* Side_Channel_Countermeasures (Section 3)
* Crypto_Algorithms

Test:
* Timing measurement (TIM-001, TIM-002)

---

### T-013: Debug Exploitation

Attack Scenario:
Attacker uses debug port to extract keys or inject code.

Mitigation:
* Debug lock after provisioning
* Authenticated debug unlock
* Rate limiting

Requirement:
* REQ-SR-DBG-001
* REQ-SR-DBG-002

Design:
* Secure_Debug_Architecture

Test:
* DBG-TEST-001

---

### T-014: Manufacturing Overproduction

Attack Scenario:
Factory produces unauthorized devices for grey market.

Mitigation:
* Batch authorization
* Backend registration
* Audit records

Requirement:
* REQ-SR-MFR-001
* REQ-SR-MFR-002

Design:
* Manufacturing_Security (Section 3)

Test:
* Factory audit

---

### T-015: Manufacturing Key Extraction

Attack Scenario:
Attacker intercepts key material during provisioning.

Mitigation:
* Encrypted provisioning channel
* Key wrapping
* Operator authentication

Requirement:
* REQ-SR-MFR-001

Design:
* Manufacturing_Security (Section 4)

Test:
* Factory audit + crypto analysis

---

## 5. Coverage Rules

* Every threat must map to at least one requirement
* Every requirement must mitigate at least one threat
* No orphan threats allowed

---

## 6. Gap Analysis

If any threat has:

* No mitigation → CRITICAL GAP
* No requirement → DESIGN GAP
* No test → VERIFICATION GAP
