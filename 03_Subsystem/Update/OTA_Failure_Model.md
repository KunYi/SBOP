# OTA Failure Model

**Document ID:** SUB-OTA-FAIL-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines all OTA failure scenarios, error codes, system response, and recovery paths. The OTA process must be resilient to network failures, storage errors, power loss, and malicious interference.

---

## 2. Failure Scenarios

### F-OTA-001: Network Interruption During Download

**Description:** TLS connection drops, network times out, or backend becomes unreachable during firmware download.

**Detection:** Socket error, TLS alert, or download timeout (configurable, default 60s).

**Response:**
1. Record bytes successfully received (checkpoint offset)
2. Close connection
3. Transition to IDLE with error status
4. Retry with exponential backoff: 1 min, 5 min, 15 min, 1 hour, 6 hours
5. After max retries (10): report failure to backend, remain on current version

**Error Code:** `ERR-OTA-DOWNLOAD-001` (Download interrupted, resumable)

**Recovery:** Resume download from last checkpoint offset using HTTP Range request.

**Test:** TEST-OTA-002

---

### F-OTA-002: Corrupted Download

**Description:** Downloaded image hash does not match expected hash.

**Detection:** SHA-256 of downloaded data ≠ expected hash from UpdateInfo.

**Response:**
1. Discard downloaded image
2. Erase inactive slot (remove partial data)
3. Transition to IDLE
4. Retry download (full restart, no resume — data is untrusted)
5. If repeated hash failures (3+): report to backend, flag as potential MITM

**Error Code:** `ERR-OTA-DOWNLOAD-002` (Hash mismatch after download)

**Recovery:** Full re-download. If persistent, backend may serve different CDN endpoint or escalate to SEV-2.

**Test:** TEST-OTA-003

---

### F-OTA-003: Verification Failure

**Description:** Downloaded image fails signature or hash verification at the OTA verification phase.

**Detection:** `crypto_verify_signature()` fails, or hash verification fails (same as boot).

**Response:**
1. Discard downloaded image
2. Erase inactive slot
3. Transition to IDLE
4. Report to backend with ERR-OTA-ACTIVATE-* error (no detailed reason)
5. Do NOT retry automatically (image is bad, not a transient error)
6. Backend may investigate: corrupted in CDN? Signing error? Attack?

**Error Code:** `ERR-OTA-ACTIVATE-001` (Signature invalid) or `ERR-OTA-ACTIVATE-002` (Hash mismatch)

**Recovery:** Backend provides corrected image. Device waits for new update notification.

**Test:** TEST-OTA-003

---

### F-OTA-004: Power Loss During Slot Write

**Description:** Power lost while writing firmware to inactive slot flash.

**Detection:** On next boot, slot metadata CRC fails → slot status is inconsistent.

**Response:**
1. Boot detects slot metadata CRC mismatch
2. Mark slot as INVALID
3. Boot from active slot (still intact)
4. After boot, OTA client detects INVALID slot
5. OTA client erases INVALID slot, restarts download from byte 0

**Error Code:** `ERR-BOOT-STOR-003` (Metadata CRC mismatch, detected at boot)

**Recovery:** Active slot was never touched — device boots normally. Full re-download of update required.

**Test:** TEST-OTA-002 (cut power at 50% write)

---

### F-OTA-005: Boot Failure After Update

**Description:** OTA installed firmware to inactive slot, slot marked PENDING, device rebooted, but new firmware failed boot verification.

**Detection:** Boot detects verification failure on PENDING slot.

**Response (automatic rollback):**
1. Boot marks PENDING slot as INVALID
2. Boot selects previous ACTIVE slot as fallback
3. Boot increments `slot_fallback_count`
4. Boot boots fallback image
5. After boot, device reports rollback to backend with error code
6. Backend increments rollback counter for this version

**Error Code:** `ERR-OTA-ROLL-001` (Rollback triggered — boot failure after update)

**Recovery:** Device runs previous known-good firmware. Backend halts deployment if rollback rate > threshold.

**Test:** TEST-OTA-004

---

### F-OTA-006: Insufficient Flash Space

**Description:** Firmware image size exceeds available space in inactive slot.

**Detection:** `image_size > available_slot_space` before download begins.

**Response:**
1. Reject update immediately
2. Transition to IDLE
3. Report to backend: `ERR-OTA-DOWNLOAD-003`

**Error Code:** `ERR-OTA-DOWNLOAD-003` (Insufficient flash space)

**Recovery:** Backend should not serve images larger than slot size. If this occurs, it's a configuration error at the backend — fix image/slot sizing.

**Test:** TEST-OTA-003

---

### F-OTA-007: Backend Authentication Failure

**Description:** Device fails to authenticate to OTA backend.

**Detection:** TLS mutual auth handshake fails, or backend returns 401/403.

**Response:**
1. Transition to IDLE
2. Retry with backoff: 5 min, 30 min, 2 hours
3. After 5 failures: log error, wait for next scheduled update check
4. If persistent: device identity may need investigation

**Error Code:** `ERR-OTA-AUTH-001` (TLS handshake failed) or `ERR-OTA-AUTH-002` (Backend rejected device identity)

**Recovery:** If caused by network: retry succeeds. If caused by device identity: factory investigation.

**Test:** TEST-OTA-001

---

## 3. Failure Principles

| # | Principle |
| --- | --- |
| 1 | **Never corrupt active firmware** — OTA only writes to inactive slot |
| 2 | **Never activate unverified image** — PENDING→ACTIVE requires boot verification |
| 3 | **Always maintain recoverable state** — active slot is never touched during OTA |
| 4 | **Resume when possible** — network interruptions should resume, not restart |
| 5 | **Report but don't leak** — error codes sent to backend must not reveal which specific byte/field failed |

---

## 4. Recovery Strategy Summary

| Scenario | Automatic Recovery | Manual Intervention |
| --- | --- | --- |
| Network interruption | Resume download (HTTP Range) | None |
| Corrupted download | Full re-download | Backend check if repeated |
| Verification failure | Wait for new update | Backend investigation |
| Power loss during write | Boot from active slot, re-download | None |
| Boot failure after update | Automatic rollback to fallback | Backend halts deployment if rate > 2% |
| Insufficient space | Reject, report | Backend config fix |
| Auth failure | Retry with backoff | Factory investigation if persistent |

---

## 5. References

| Document | Reference |
| --- | --- |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §6 |
| OTA Flow Pseudocode | `OTA_Flow_Pseudocode.md` |
| OTA Recovery | `OTA_Recovery.md` |
| OTA Interface | `OTA_Interface.md` |
| Boot Failure Model | `../../03_Subsystem/Boot/Boot_Failure_Model.md` |
| Incident Response | `../../06_Operations/Incident_Response.md` |
| OTA Deployment | `../../06_Operations/OTA_Deployment.md` |
