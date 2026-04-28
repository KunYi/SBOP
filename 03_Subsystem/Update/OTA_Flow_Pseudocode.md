# OTA Flow Pseudocode Specification

**Document ID:** SUB-OTA-FLOW-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document specifies the concrete OTA update algorithm in pseudocode form. It defines the update flow, rollback procedure, recovery from interruption, and integration with the boot subsystem.

→ See `03_Subsystem/Update/OTA_State_Machine.md` for state definitions.
→ See `03_Subsystem/Update/OTA_Failure_Model.md` for failure mode analysis.

---

## 2. Main OTA Update Flow

```
function ota_perform_update(check_result: UpdateCheckResult) -> Result<(), OTAError>:
    // ── Phase 0: Pre-Flight Checks ──
    state = OTA_IDLE
    
    // Verify that update is available and authorized
    match check_result:
        case NoUpdateAvailable:
            return OK  // Nothing to do
        case UpdateAvailable(update):
            proceed
        case Error(e):
            return Err(e)
    
    // Check device state
    assert(battery_pct >= update.min_battery_pct, ERR-OTA-DOWNLOAD-003)
    assert(storage_has_inactive_slot(), ERR-OTA-DOWNLOAD-003)
    
    // ── Phase 1: Authenticate with Backend ──
    state = OTA_AUTHENTICATE
    let device_id = identity_get_device_id()
    assert(device_id.is_ok(), ERR-ID-PROV-001)
    
    let challenge = crypto_random_bytes(32)
    let auth_result = ota_authenticate(device_id.unwrap(), challenge.as_slice())
    assert(auth_result.is_ok(), ERR-OTA-AUTH-001)
    let auth_token = auth_result.unwrap()
    
    // Verify token is fresh and not expired
    assert(auth_token.expires_at > current_timestamp(), ERR-OTA-AUTH-001)
    
    // ── Phase 2: Check for Available Update ──
    state = OTA_CHECK
    let update_info = backend_request_update(device_id.unwrap(), current_version, auth_token)
    
    match update_info:
        case Ok(NoUpdate):
            return OK  // No update available after auth
        case Ok(UpdateAvailable(info)):
            proceed
        case Err(ERR-OTA-AUTH-003):
            return Err(ERR-OTA-AUTH-003)  // Not authorized; policy or device blocked
        case Err(e):
            return Err(e)
    
    // Verify update info integrity
    let sig_valid = crypto_verify_signature(
        data = serialize_update_metadata(info),
        sig = info.signature.signature_data,
        sig_len = info.signature.signature_size,
        sig_type = info.signature.signature_type,
        key = key_resolve(BACKEND_AUTH)
    )
    assert(sig_valid == true, ERR-OTA-AUTH-001)
    
    // ── Phase 3: Download Firmware ──
    state = OTA_DOWNLOAD
    let inactive_slot = storage_get_inactive_slot()
    assert(inactive_slot != SLOT_INVALID, ERR-OTA-DOWNLOAD-003)
    
    // Erase target slot if needed
    if storage_get_slot_status(inactive_slot) != EMPTY:
        storage_erase_slot(inactive_slot)?
    
    // Download with hash verification per chunk
    let download_handle = http_download_begin(info.download_url)
    let running_hash = hash_init(SHA256)
    let bytes_written = 0
    
    loop:
        let chunk = http_download_chunk(download_handle, CHUNK_SIZE)
        match chunk:
            case Data(data):
                running_hash = hash_update(running_hash, data)
                storage_write(inactive_slot_base + bytes_written, data)?
                bytes_written += data.len
                
                // Verify chunk if chunk hashes are provided
                if info.chunk_hashes is available:
                    let chunk_hash = crypto_compute_hash(data)
                    let expected = info.chunk_hashes[current_chunk_index]
                    assert(constant_time_compare(chunk_hash, expected, 32), ERR-OTA-DOWNLOAD-002)
                
            case End:
                break
            case Error(e):
                http_download_abort(download_handle)
                return Err(ERR-OTA-DOWNLOAD-001)
    
    let final_hash = hash_finalize(running_hash)
    
    // Verify full image hash
    assert(constant_time_compare(final_hash, info.image_hash, 32), ERR-OTA-DOWNLOAD-002)
    assert(bytes_written == info.image_size, ERR-OTA-DOWNLOAD-002)
    
    // ── Phase 4: Verify Downloaded Image ──
    state = OTA_VERIFY
    
    // Read back and verify signature
    let image_data = storage_read_slot(inactive_slot)
    let header = parse_header(image_data)
    assert(header.is_ok(), ERR-BOOT-PARSE-001)
    
    let header = header.unwrap()
    let payload = image_data[header.header_size .. header.header_size + header.image_size]
    let sig_block = parse_signature_block(image_data, header)
    assert(sig_block.is_ok(), ERR-BOOT-PARSE-006)
    
    let image_sig_valid = crypto_verify_signature(
        data = image_data[0 .. header.header_size + header.image_size],
        sig = sig_block.unwrap().signature_data,
        sig_len = sig_block.unwrap().signature_size,
        sig_type = sig_block.unwrap().signature_type,
        key = key_resolve(IMAGE_VERIFY)
    )
    assert(image_sig_valid == true, ERR-BOOT-CRYPTO-001)
    
    // ── Phase 5: Store and Mark Pending ──
    state = OTA_INSTALL
    
    // Set slot metadata
    storage_set_slot_status(inactive_slot, VERIFIED)?
    storage_set_slot_version(inactive_slot, info.firmware_version)?
    
    // Verify slot was written correctly
    let verify_hash = crypto_compute_hash(storage_read_slot(inactive_slot).payload())
    assert(constant_time_compare(verify_hash, info.image_hash, 32), ERR-OTA-INSTALL-002)
    
    // ── Phase 6: Activate ──
    state = OTA_ACTIVATE
    
    // Set pending slot for boot
    boot_set_pending_image(inactive_slot)?
    
    // Report success to backend
    backend_report_update_installed(device_id.unwrap(), info.firmware_version)
    
    // Trigger system reset to activate new firmware
    system_reset()
    
    // Does not return
```

---

## 3. Rollback Procedure

```
function ota_rollback(reason: OTAError) -> Result<(), OTAError>:
    state = OTA_ROLLBACK
    
    // Log rollback event
    rollback_log_write(reason)
    
    // Find the previous valid slot
    let fallback_slot = storage_get_fallback_slot()
    assert(fallback_slot != SLOT_INVALID, ERR-OTA-ACTIVATE-001)
    
    // Verify fallback is still valid before switching
    let fallback_check = boot_verify_image_metadata(fallback_slot)
    assert(fallback_check.is_ok(), ERR-OTA-ACTIVATE-001)
    
    // Mark failed slot as invalid
    let failed_slot = storage_get_active_slot()
    storage_set_slot_status(failed_slot, INVALID)
    
    // Switch to fallback
    boot_set_pending_image(fallback_slot)?
    
    // Report rollback to backend
    backend_report_rollback(device_id, reason, failed_version, fallback_version)
    
    // Increment rollback counter
    storage_increment_rollback_counter()
    
    // Reset to activate fallback
    system_reset()
```

---

## 4. Recovery from Interrupted Update

```
function ota_recover_from_interruption() -> Result<SlotStatus, OTAError>:
    // Called at boot when an interrupted OTA is detected
    
    let inactive_slot = storage_get_inactive_slot()
    let slot_status = storage_get_slot_status(inactive_slot)
    
    match slot_status:
        case VERIFIED:
            // Image was fully downloaded and verified, but not activated
            // This is a pre-activation interruption (e.g., power loss before reset)
            // The image is verified; safe to proceed with activation
            boot_set_pending_image(inactive_slot)
            storage_set_slot_status(inactive_slot, PENDING)
            backend_report_recovery("OTA_INSTALL_RECOVERED")
            system_reset()
        
        case PENDING:
            // Image was being written; likely corrupted
            // Discard and mark slot as invalid
            storage_erase_slot(inactive_slot)
            storage_set_slot_status(inactive_slot, EMPTY)
            backend_report_recovery("OTA_INSTALL_CORRUPTED")
            return OK  // Boot will continue with active slot
        
        case ACTIVE:
            // No interrupted OTA; this slot is the active one
            return OK
        
        case EMPTY, INVALID, CORRUPT, FALLBACK:
            // No pending OTA in this slot
            return OK
    
    return OK
```

---

## 5. OTA State Transition Table

| Current State | Event / Condition | Next State | Action |
| --- | --- | --- | --- |
| IDLE | Update check: update available | AUTHENTICATE | Request auth |
| IDLE | Update check: no update | IDLE | Wait for next check |
| AUTHENTICATE | Auth success | CHECK | Request update info |
| AUTHENTICATE | Auth failure (retryable) | IDLE | Backoff and retry |
| AUTHENTICATE | Auth failure (terminal) | IDLE | Report to backend; abort |
| CHECK | Update authorized | DOWNLOAD | Start download |
| CHECK | Update denied | IDLE | Policy-based; no action |
| DOWNLOAD | Chunk received | DOWNLOAD | Write chunk; update hash |
| DOWNLOAD | Download complete + hash OK | VERIFY | Verify image |
| DOWNLOAD | Download complete + hash mismatch | IDLE | Discard; report; may retry |
| DOWNLOAD | Connection lost | IDLE | Discard partial; retry with resume |
| VERIFY | Signature valid | INSTALL | Mark slot VERIFIED |
| VERIFY | Signature invalid | IDLE | Discard image; report |
| INSTALL | Slot written OK | ACTIVATE | Set pending; reset |
| INSTALL | Slot write error | ROLLBACK | Revert to previous |
| ACTIVATE | Boot success | IDLE | New firmware running |
| ACTIVATE | Boot failure | ROLLBACK | Enter rollback procedure |
| ROLLBACK | Fallback exists | ACTIVATE (fallback) | Switch to fallback |
| ROLLBACK | No fallback | FAILSAFE | Wait for OTA recovery |

---

## 6. Guard Conditions

| Transition | Guard |
| --- | --- |
| IDLE → AUTHENTICATE | update_available AND connectivity OK AND battery >= min_pct |
| AUTHENTICATE → CHECK | auth_token valid AND auth_token not expired |
| DOWNLOAD → VERIFY | downloaded_bytes == image_size AND SHA-256 matches |
| VERIFY → INSTALL | ECDSA/Ed25519 signature valid |
| INSTALL → ACTIVATE | slot write verified by read-back + hash check |
| ACTIVATE → IDLE | Boot reports successful execution of new image |
| ACTIVATE → ROLLBACK | Boot reports failure or fails to report within timeout |

---

## 7. OTA Timing Budgets

| Phase | Duration | Notes |
| --- | --- | --- |
| AUTHENTICATE | < 5 s | Network-dependent |
| CHECK | < 2 s | Backend response time |
| DOWNLOAD | Variable | Depends on image size and bandwidth; supports resume |
| VERIFY | < 150 ms | Signature + hash over full image |
| INSTALL | < 500 ms | Flash write (depends on image size) |
| ACTIVATE | < 300 ms | Reset + boot verification |
| ROLLBACK | < 300 ms | Reset + fallback boot |

---

## 8. Error Recovery Strategy

| Error | Strategy |
| --- | --- |
| ERR-OTA-AUTH-001 | Retry with exponential backoff (1s, 2s, 4s, 8s); max 5 retries |
| ERR-OTA-DOWNLOAD-001 | Resume from last received chunk (HTTP Range) |
| ERR-OTA-DOWNLOAD-002 | Discard; restart download from beginning |
| ERR-OTA-DOWNLOAD-003 | Abort; wait for storage availability or battery charge |
| ERR-OTA-INSTALL-001/002 | Retry write (max 3 attempts); then mark slot bad; try other slot |
| ERR-OTA-ACTIVATE-001/002 | Automatic rollback to fallback slot |
