# Boot Failure Model

**Document ID:** SUB-BOOT-FAIL-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines all boot failure scenarios, their error codes, system response, and recovery paths. Each failure maps to a specific error code in `Error_Code_Catalog.md` and a defined transition to FAILSAFE or fallback.

---

## 2. Failure Scenarios

### F-BOOT-001: Metadata Corruption

**Description:** Slot metadata CRC validation fails for the selected slot.

**Detection:** CRC-32 mismatch on `SlotMetadata` structure read from flash.

**Response:**
1. Mark slot as INVALID (if writable)
2. Attempt fallback to other slot
3. If both slots invalid → FAILSAFE

**Error Code:** `ERR-STOR-READ-001` (Metadata CRC mismatch — storage read failure)

**Recovery:** Backend-initiated OTA to repair corrupted slot.

**Test:** TEST-BOOT-003

---

### F-BOOT-002: Signature Verification Failure

**Description:** Ed25519 signature verification fails for the loaded image.

**Detection:** Cryptographic signature verification returns false or error.

**Response:**
1. Log failure (no detailed reason — avoid oracle)
2. Mark slot as INVALID
3. Attempt fallback to other slot
4. If fallback also fails signature → FAILSAFE

**Error Code:** `ERR-BOOT-CRYPTO-001` (Signature verification failed)

**Recovery:** If fallback succeeds, report rollback to backend. If FAILSAFE, await backend recovery OTA.

**Test:** TEST-BOOT-002

---

### F-BOOT-003: Integrity Check Failure

**Description:** SHA-256 hash of image payload does not match expected hash.

**Detection:** Computed hash ≠ hash in ImageHeader after constant-time comparison.

**Response:**
1. Mark slot as INVALID
2. Attempt fallback to other slot
3. If fallback also fails → FAILSAFE

**Error Code:** `ERR-BOOT-CRYPTO-002` (Hash mismatch)

**Recovery:** Same as F-BOOT-002. Hash failure may indicate flash degradation (SEU/TID) or corruption.

**Test:** TEST-BOOT-003

---

### F-BOOT-004: Version Rollback Detected

**Description:** Firmware version ≤ current monotonic version counter.

**Detection:** Redundant read of version counter from OTP; comparison with new firmware version.

**Response:**
1. Reject image (do NOT mark slot INVALID — image was validly signed, just too old)
2. Attempt fallback to other slot
3. If both slots have old versions → FAILSAFE (unlikely — requires both slots rolled back)

**Error Code:** `ERR-BOOT-VERSION-001` (Version too old / rollback rejected)

**Recovery:** Update to a newer firmware version via OTA. The device will accept firmware with version > counter.

**Test:** TEST-BOOT-004, FI-004

---

### F-BOOT-005: Undefined State Transition

**Description:** Boot state machine enters an undefined state or attempts an invalid transition.

**Detection:** Phase tracking flags indicate a phase was skipped, or state variable contains invalid value.

**Response:**
1. Immediate FAILSAFE
2. Log the invalid state in tamper log
3. Treat as potential fault injection attempt

**Error Code:** `ERR-BOOT-STATE-001` (Invalid state transition)

**Recovery:** On next power cycle, state machine re-initializes to RESET. If caused by SEU, reboot may resolve. If persistent, backend recovery.

**Test:** FI-001 (fault injection to skip phase)

---

### F-BOOT-006: Crypto Engine Hardware Fault

**Description:** Hardware crypto accelerator reports an error or is unresponsive.

**Detection:** Crypto API returns hardware fault error.

**Response:**
1. Retry with software crypto fallback (if available)
2. If fallback also fails → FAILSAFE
3. Log hardware fault for diagnostics

**Error Code:** `ERR-BOOT-CRYPTO-006` (Crypto engine hardware fault)

**Recovery:** Persistent crypto fault may indicate hardware failure. Device requires replacement if both HW and SW crypto fail.

**Test:** FI-001 (inject fault during crypto operation)

---

### F-BOOT-007: Tamper Event During Boot

**Description:** Tamper sensor triggers during boot (enclosure, voltage glitch, clock anomaly).

**Detection:** `sample_tamper_sensors()` returns TAMPER_MAJOR or TAMPER_CRITICAL.

**Response:**
- TAMPER_MAJOR: Log event, enter FAILSAFE
- TAMPER_CRITICAL: Zeroize keys, enter TAMPER_LOCK (irreversible)

**Error Code:** `ERR-HW-TAMPER-001` (Tamper event during boot)

**Recovery:** FAILSAFE: await backend recovery. TAMPER_LOCK: device irrecoverable (keys destroyed).

**Test:** FI-001, Physical pentest

---

## 3. Failure Principles

| # | Principle |
| --- | --- |
| 1 | **Fail closed** — never execute unverified firmware under any condition |
| 2 | **No recovery inside Boot** — Boot does not attempt self-repair; recovery is the Update subsystem's responsibility |
| 3 | **Fallback before FAILSAFE** — always attempt the other slot before giving up |
| 4 | **Log minimal information** — don't expose diagnostic data that creates an oracle for attackers |
| 5 | **Constant-time failure** — failure path timing must equal success path timing |

---

## 4. Observability

The boot process exposes the following for diagnostic purposes:

| Information | Exposure | Constraint |
| --- | --- | --- |
| Last boot error code | Telemetry (anonymized) | ERR-BOOT-* code only, no details |
| Boot state at failure | Tamper log (internal) | Not exposed to Zone 2 |
| Slot status | Telemetry | A/B/INVALID status only |
| Boot duration | Telemetry | Value only, no timing-profile detail |

The following MUST NOT be exposed:
- Partial hash values
- Signature verification intermediate results
- Which byte or field caused the failure
- Key material of any kind

---

## 5. References

| Document | Reference |
| --- | --- |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §4 |
| Boot Flow Pseudocode | `Boot_Flow_Pseudocode.md` |
| Boot Interface | `Boot_Interface.md` |
| Fault Injection Test | `../../05_Verification/Fault_Injection_Test.md` |
| Physical Tamper Resistance | `../../04_Security/Physical_Tamper_Resistance.md` |
