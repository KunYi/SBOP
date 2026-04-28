# OTA Recovery and Rollback Strategy

**Document ID:** SUB-OTA-REC-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines recovery mechanisms ensuring system survivability under all OTA failure conditions. The recovery strategy guarantees that at least one valid firmware image always exists and the device never enters an unrecoverable (bricked) state.

---

## 2. Dual Image Model

| Slot | Purpose | State |
|------|---------|-------|
| Slot A | Primary firmware image | ACTIVE, PENDING, or INVALID |
| Slot B | Secondary firmware image | ACTIVE, PENDING, or INVALID |

Rules:
- At most one slot is ACTIVE at any time
- Updates always write to the inactive slot
- The active slot is never overwritten during an update
- After successful update confirmation, the previously active slot becomes inactive

---

## 3. Activation Flow

```
1. OTA downloads and verifies new image
2. OTA writes image to INACTIVE slot
3. OTA marks slot as PENDING
4. OTA requests reboot
5. Boot selects PENDING slot
6. Boot performs full verification chain (phases 1-12)
7. Boot executes new image (EXECUTE)
8. Application confirms successful boot (explicit confirmation)
9. Boot marks PENDING → ACTIVE, previous ACTIVE → INACTIVE
```

If step 8 never occurs (confirmation timeout), slot remains PENDING and rollback is triggered.

---

## 4. Rollback Conditions

| Condition | Detection | Action |
|-----------|-----------|--------|
| Boot verification failure | Boot phases 2-12 before EXECUTE | FAILSAFE — try other slot |
| New image crashes after boot | Watchdog timeout | Reset — Boot selects previous ACTIVE |
| Confirmation timeout | Application fails to confirm within N seconds | Boot marks PENDING slot INVALID |
| Repeated boot failures | Counter exceeds threshold | Boot selects known-good slot, reports to backend |
| OTA stage failure | Flash write error, verification mismatch | Mark slot INVALID, report error |

---

## 5. Rollback Mechanism

```
Boot finds PENDING slot with version V_new:
  │
  ├── Verify V_new (signature, integrity, version check)
  │     │
  │     ├── PASS → EXECUTE V_new
  │     │            │
  │     │            └── Wait for confirmation (watchdog armed, N seconds)
  │     │                   │
  │     │                   ├── Confirmed → Mark V_new ACTIVE, V_old INACTIVE
  │     │                   └── Timeout → Mark V_new INVALID → Reboot → Select V_old
  │     │
  │     └── FAIL → Mark V_new INVALID → Select V_old
  │
  └── Repeat for V_old
        │
        ├── PASS → EXECUTE V_old (recovery successful)
        └── FAIL → FAILSAFE (no valid image — requires backend intervention)
```

---

## 6. Confirmation Protocol

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Confirmation type | Explicit (application must call `confirm_boot()`) | Prevents implicit trust of untested image |
| Confirmation deadline | Configurable (default: 60s) | Long enough for application init + self-test |
| Confirmation gate | Application must pass self-tests before calling `confirm_boot()` | Prevents confirmation of broken image |
| Fallback on timeout | Slot marked INVALID, boot reverts | Safe default — no confirmation = no trust |

---

## 7. Watchdog Integration

| Watchdog Stage | Timeout | Action |
|----------------|---------|--------|
| Boot (phases 1-12) | Tight (1-5s) | Reset → retry boot |
| Post-EXECUTE (confirmation window) | Medium (60s) | Reset → boot detects unconfirmed PENDING → mark INVALID → rollback |
| Runtime (after confirmation) | Normal (application-managed) | Reset → boot to confirmed ACTIVE slot |

---

## 8. Safety Guarantees

| # | Guarantee | Enforcement |
|---|-----------|-------------|
| 1 | Always one valid image | Both slots never invalidated simultaneously |
| 2 | No permanent bricking | Failsafe recovery loop: always try both slots |
| 3 | Atomic slot state transitions | Slot state written atomically; interrupted write detected |
| 4 | No rollback bypass | Confirmation is cryptographic — cannot be spoofed |
| 5 | Backend-aware recovery | Device reports recovery events to backend for fleet monitoring |
| 6 | Version monotonicity preserved | OTP counter not burned until confirmation success (or burned pre-boot, pre-execute) |

---

## 9. Recovery Scenarios

| Scenario | Recovery Path | Automatic? |
|----------|--------------|------------|
| Download interrupted (network) | Resume via Range request | Yes |
| Verification failure (bad image) | Discard; re-download from backend | Yes |
| Flash write error | Mark slot INVALID; retry or wait | Yes |
| Boot crash (new image) | Watchdog → rollback to previous | Yes |
| Confirmation timeout | Mark INVALID → rollback | Yes |
| Both slots invalid | FAILSAFE — await recovery image via minimal bootloader | No (requires service tool) |
| Tamper detected | Key zeroization → TAMPER_LOCKED | No (permanent) |

---

## 10. References

| Document | Reference |
|----------|-----------|
| OTA Overview | `OTA_Overview.md` |
| OTA Flow Pseudocode | `OTA_Flow_Pseudocode.md` |
| OTA Failure Model | `OTA_Failure_Model.md` |
| Boot Flow Pseudocode | `../Boot/Boot_Flow_Pseudocode.md` |
| Trust Model | `../../00_Architecture/Trust_Model.md` |
