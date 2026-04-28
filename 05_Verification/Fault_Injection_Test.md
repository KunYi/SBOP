# Fault Injection Test Plan

**Document ID:** VER-FI-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines tests to validate system robustness against fault injection attacks. Validates physical tamper resistance claims (C5) and security requirements REQ-SR-PHY-001, REQ-SR-PHY-002.

---

## 2. Fault Model

| Fault Type | Mechanism | Target | SL Required |
|------------|-----------|--------|-------------|
| Instruction skip | Voltage glitch on clock edge | Conditional branches, security checks | SL 3+ |
| Data corruption | EM pulse during memory access | Hash values, signatures, version counters | SL 3+ |
| Timing disruption | Clock glitch, overclocking | State machine transitions | SL 2+ |
| Register corruption | Voltage glitch during register read | Configuration registers, MPU settings | SL 3+ |
| Unexpected reset | Power brown-out | State persistence, OTP writes | SL 2+ |

---

## 3. Test Equipment

| Tool | Purpose | Specification |
|------|---------|---------------|
| ChipWhisperer | Voltage glitching | < 10 ns glitch width, 0-3.3V range |
| Clock glitcher | Clock manipulation | < 1 ns jitter |
| EM pulse generator | EM fault injection | Localized EM probe |
| Oscilloscope | Signal monitoring | 1 GHz+ bandwidth |
| Power supply (programmable) | Brown-out simulation | 0-5V, < 1 mV resolution |

---

## 4. Test Scenarios

### FI-001: Skip Signature Verification

| Parameter | Value |
|-----------|-------|
| **Target** | Conditional branch after signature verification |
| **Method** | Voltage glitch on CPU clock edge |
| **Glitch parameters** | Width: 5-50 ns, Voltage: 0.5-2.0V below nominal, Timing: aligned to verification result read |
| **Iterations** | 1000 per parameter set |
| **Setup** | Device booting with valid image. Inject glitch at VERIFY_SIGNATURE → VERIFY_INTEGRITY transition. |
| **Expected result** | Device enters FAILSAFE — execution never reaches Zone 2. Glitch must not cause silent skip to EXECUTE. |
| **Pass criteria** | 0 successful boots with invalid/missing signature verification. 100% FAILSAFE or detectable fault. |
| **Attack tree** | A1 (Signature Forgery), A7 (Physical Tamper) |

### FI-002: Corrupt Firmware Hash

| Parameter | Value |
|-----------|-------|
| **Target** | SHA-256 hash comparison result |
| **Method** | EM pulse during `constant_time_compare()` execution |
| **Glitch parameters** | Pulse width: 10-100 ns, Field strength: calibrated per device, Timing: aligned to comparison loop |
| **Iterations** | 500 per target area |
| **Setup** | Device booting with valid image. Inject EM pulse during hash comparison. |
| **Expected result** | Hash mismatch detected — device enters FAILSAFE. Glitch must not flip comparison result to "match" when hashes differ. |
| **Pass criteria** | 0 false "match" results. All glitched comparisons result in detected mismatch or FAILSAFE. |
| **Attack tree** | A3 (Image Tampering), A7 (Physical Tamper), A8 (Side-Channel) |

### FI-003: Interrupt OTA Write

| Parameter | Value |
|-----------|-------|
| **Target** | Flash write to inactive slot during OTA STAGE |
| **Method** | Power brown-out during flash write cycle |
| **Glitch parameters** | V_drop: 20-80% below nominal, Duration: 1-100 ms, Timing: mid-write |
| **Iterations** | 100 per timing offset |
| **Setup** | OTA in STAGE phase. Trigger brown-out at various points during flash write. |
| **Expected result** | Active slot (running image) unchanged. Inactive slot marked invalid. Boot recovers to active slot. |
| **Pass criteria** | Active image never corrupted. Device boots successfully to pre-OTA image. No intermediate state exploitable. |
| **Attack tree** | A3, A7, A11 (Zone Hopping via shared flash) |

### FI-004: Glitch During Version Check

| Parameter | Value |
|-----------|-------|
| **Target** | Version comparison (`if image_version >= otp_version`) |
| **Method** | Voltage glitch on branch instruction |
| **Glitch parameters** | Width: 5-100 ns, Voltage: 0.3-1.8V below nominal, Timing: aligned to comparison branch |
| **Iterations** | 2000 per parameter set |
| **Setup** | Device with OTP counter = v5. Inject image with version = v3 (rollback). Inject glitch at version comparison. |
| **Expected result** | Rollback detected — device enters FAILSAFE. Glitch must not bypass version check to allow v3 execution when v5 is minimum. |
| **Pass criteria** | 0 successful boots with rollback version. Redundant version checks must also trigger. |
| **Attack tree** | A4 (Rollback), A7 (Physical Tamper) |

---

## 5. Additional Scenarios

| Test ID | Target | Description |
|---------|--------|-------------|
| FI-005 | MPU configuration | Glitch during MPU register write at LOCK_BOOT |
| FI-006 | OTP write | Glitch during COMMIT_VERSION OTP burn |
| FI-007 | Debug lock | Glitch during debug lifecycle state transition |
| FI-008 | Slot selection | Glitch during SELECT_SLOT validity check |

---

## 6. Test Environment

| Environment | Purpose |
|-------------|---------|
| Reference platform | Initial characterization, parameter sweep |
| Target hardware | Final verification with exact silicon |
| Temperature chamber | Repeat FI-001..004 at -40°C, +25°C, +85°C (or per deployment profile) |
| Shielded room | EM fault injection (FI-002) |

---

## 7. Acceptance Criteria

- No fault may bypass security checks — all must result in FAILSAFE or detectable failure
- All faults must lead to safe state — no undefined behavior
- Redundant checks must catch single-glitch bypass attempts (SL 3+)
- Results must be reproducible across temperature range
- No silent failures — every injected fault must be observable (FAILSAFE, watchdog, or tamper flag)

---

## 8. References

| Document | Reference |
|----------|-----------|
| Physical Tamper Resistance | `../04_Security/Physical_Tamper_Resistance.md` |
| Security Controls | `../04_Security/Security_Controls.md` |
| Test Strategy | `Test_Strategy.md` |
| Test Cases | `Test_Cases.md` |
| Attack Tree | `../04_Security/Attack_Tree.md` (A7, A8) |
