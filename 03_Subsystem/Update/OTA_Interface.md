# OTA Interface Specification

**Document ID:** SUB-OTA-IF-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the abstract interfaces required by the OTA Update subsystem for backend communication, slot management, identity retrieval, and storage access.

→ Formal contracts in `../Interface_Contracts.md` §5 (OTA), §6 (identity), §4 (crypto)

---

## 2. Backend Communication Interface

### 2.1 ota_authenticate

```
Function: ota_authenticate() → Result<AuthToken, OTAError>

Purpose: Authenticate device to OTA backend using mutual TLS with device certificate (KD_Auth-derived).

Preconditions:
  - TLS stack initialized with device client certificate
  - Backend URL configured
  - Network connectivity available

Postconditions:
  - Ok(token) → device authenticated; token valid for session duration
  - Err(OTAError::AuthFailed) → authentication rejected
  - Err(OTAError::NetworkError) → connectivity failure

Errors:
  - ERR-OTA-AUTH-001: TLS handshake failed
  - ERR-OTA-AUTH-002: Backend rejected device identity
  - ERR-OTA-AUTH-003: Auth token expired or invalid
```

### 2.2 request_update

```
Function: request_update(token: &AuthToken) → Result<UpdateInfo, OTAError>

Purpose: Query backend for available firmware update matching device eligibility.

Preconditions:
  - token is valid and not expired
  - Device identity is registered and in good standing

Postconditions:
  - Ok(info) → UpdateInfo contains: version, download_url, image_size, image_hash, signature
  - Ok(info) where info.version == current_version → no update available
  - Err(OTAError::NoUpdateAvailable) → device already at latest version

Errors:
  - ERR-OTA-UPDATE-001: No update available for this device
  - ERR-OTA-UPDATE-002: Device blocked from updates
  - ERR-OTA-UPDATE-003: Update metadata malformed
```

### 2.3 download_firmware

```
Function: download_firmware(url: &str, expected_hash: &[u8; 32], image_size: u32, slot: SlotID) → Result<(), OTAError>

Purpose: Download firmware image via HTTPS with chunked transfer and resume support.

Preconditions:
  - slot is INACTIVE or EMPTY
  - Sufficient flash space in target slot
  - Network connectivity available

Postconditions:
  - Ok(()) → image written to slot, write-verified
  - Image hash matches expected_hash
  - Partial download on error → can resume

Errors:
  - ERR-OTA-DOWNLOAD-001: Download interrupted (resumable)
  - ERR-OTA-DOWNLOAD-002: Hash mismatch after download
  - ERR-OTA-DOWNLOAD-003: Insufficient flash space
  - ERR-OTA-DOWNLOAD-004: TLS error during download
  - ERR-OTA-DOWNLOAD-005: Image size mismatch
```

---

## 3. Slot Management Interface

### 3.1 set_pending_slot

```
Function: set_pending_slot(slot: SlotID) → Result<(), OTAError>

Purpose: Mark the downloaded slot as PENDING activation. This triggers reboot into the new image.

Preconditions:
  - slot contains a fully downloaded and verified image
  - slot status is INACTIVE
  - Boot subsystem is ready for slot transition

Postconditions:
  - Ok(()) → slot.status = PENDING; next boot will attempt this slot
  - Slot metadata CRC updated
  - Operation is atomic (survives power loss)

Errors:
  - ERR-OTA-SLOT-001: Slot status transition failed
  - ERR-OTA-SLOT-002: Slot metadata write failed
```

### 3.2 confirm_boot_success

```
Function: confirm_boot_success() → Result<(), OTAError>

Purpose: Called by application after successful boot of new image. Confirms the PENDING → ACTIVE transition was successful.

Postconditions:
  - Current slot.status = ACTIVE
  - Previous ACTIVE slot demoted to INACTIVE (fallback slot)
```

### 3.3 trigger_rollback

```
Function: trigger_rollback() → Result<(), OTAError>

Purpose: Explicitly trigger rollback to previous firmware version. Called when application detects critical error after update.

Postconditions:
  - Current slot marked INVALID
  - System reboots into fallback slot
```

### 3.4 read_slot_status

```
Function: read_slot_status() → Result<(SlotStatus, SlotStatus), OTAError>

Purpose: Read the status of both slots.

Postconditions:
  - Ok((slot_a_status, slot_b_status)) → current slot states
```

---

## 4. Storage Interface

### 4.1 write_inactive_slot

```
Function: write_inactive_slot(data: &[u8], offset: u32) → Result<(), OTAError>

Purpose: Write a chunk of firmware data to the inactive slot at the given offset.

Preconditions:
  - Inactive slot is writable (via HAL)
  - offset + data.len() ≤ slot_size
  - Boot subsystem allows write to inactive slot only

Postconditions:
  - Ok(()) → data written to slot flash at offset
  - Write-verify performed: data read back and compared

Errors:
  - ERR-OTA-STOR-001: Flash write failed
  - ERR-OTA-STOR-002: Write verification failed (readback mismatch)
  - ERR-OTA-STOR-003: Offset out of bounds
```

### 4.2 erase_slot

```
Function: erase_slot(slot: SlotID) → Result<(), OTAError>

Purpose: Erase the specified slot before writing new firmware.

Preconditions:
  - slot is INACTIVE or EMPTY or INVALID
  - slot is NOT the currently ACTIVE slot

Postconditions:
  - Ok(()) → slot flash erased, slot.status = EMPTY
```

---

## 5. Identity Interface

### 5.1 get_device_identity

```
Function: get_device_identity() → Result<DeviceID, OTAError>

Purpose: Get device identity for backend authentication and update eligibility.

Postconditions:
  - Ok(id) → id used for client certificate and backend identification
```

---

## 6. Recovery Interface

### 6.1 ota_rollback

```
Function: ota_rollback(reason: OTAError) → Result<(), OTAError>

Purpose: Execute OTA rollback sequence after failed update or boot.

Algorithm:
  1. Mark current (failed) slot as INVALID
  2. Switch to fallback slot
  3. Report rollback to backend with reason
  4. Reboot into fallback slot
```

### 6.2 ota_recover_from_interruption

```
Function: ota_recover_from_interruption() → Result<OTAState, OTAError>

Purpose: Determine OTA state after power-cycle during update and resume or restart.

Postconditions:
  - Returns current OTA state (DOWNLOADING, VERIFYING, etc.)
  - Download resumes from last confirmed offset
  - Corrupted slot is erased and download restarted if needed
```

---

## 7. Constraints

| ID | Constraint |
| --- | --- |
| C-OTA-01 | All write operations must be idempotent (retry-safe) |
| C-OTA-02 | Partial slot writes must be detectable (CRC mismatch → INVALID) |
| C-OTA-03 | Active slot must never be writable via OTA interface |
| C-OTA-04 | No raw key material in OTA messages (key handles only) |
| C-OTA-05 | Download must verify TLS server certificate (no bypass) |
| C-OTA-06 | Image must be fully downloaded AND verified before slot transition to PENDING |
| C-OTA-07 | OTA download must not block safety-critical application functions |
| C-OTA-08 | All backend communication requires mutual TLS (device client certificate) |

---

## 8. References

| Document | Reference |
| --- | --- |
| Formal Interface Contracts | `../Interface_Contracts.md` §5-6 |
| OTA Flow Pseudocode | `OTA_Flow_Pseudocode.md` |
| OTA Recovery | `OTA_Recovery.md` |
| Data Structures | `../../02_System_Design/Data_Structures.md` §7-9 |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §6 |
