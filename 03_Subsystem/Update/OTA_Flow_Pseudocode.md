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
    
    // ── Phase 3: Download Firmware with Progress Journal ──
    // Downloads the OTA Image Package (encrypted transport format):
    //   OTAHeader || AAD || AES-256-GCM(ImageHeader || Firmware || SignatureBlock) || GCM_Tag || Ed25519_Sig
    // The OTA layer stores the package as-is. The bootloader decrypts it on
    // first boot (FLAG_OTA_PENDING → Phase 4.5 ECDH decrypt + KD_Storage re-encrypt).
    state = OTA_DOWNLOAD
    let inactive_slot = storage_get_inactive_slot()
    assert(inactive_slot != SLOT_INVALID, ERR-OTA-DOWNLOAD-003)
    
    // Erase target slot if needed
    if storage_get_slot_status(inactive_slot) != EMPTY:
        storage_erase_slot(inactive_slot)?
    
    // Initialize progress journal for resume-on-interrupt
    let progress = ota_progress_init(inactive_slot, info.image_size, info.image_hash)
    
    // Check if a previous partial download can be resumed
    let prev_progress = ota_progress_read_last()
    let bytes_written = 0
    let running_hash = hash_init(SHA256)
    
    if prev_progress.valid
       and prev_progress.target_slot == inactive_slot
       and prev_progress.expected_hash == info.image_hash:
        // Resume from checkpoint: verify existing data integrity
        let existing_data = storage_read_slot_range(inactive_slot, 0, prev_progress.bytes_written)
        let existing_hash = crypto_compute_hash(existing_data)
        let truncated = existing_hash[0..8]  // First 8 bytes of SHA-256
        if constant_time_compare(truncated, prev_progress.running_hash, 8):
            // Existing data valid — resume from where we left off
            bytes_written = prev_progress.bytes_written
            running_hash = hash_init(SHA256)
            running_hash = hash_update(running_hash, existing_data)
            log("OTA resume: {} bytes already downloaded", bytes_written)
        else:
            // Existing data corrupt — restart from beginning
            storage_erase_slot(inactive_slot)?
            ota_progress_clear()
            log("OTA resume failed: data corrupt, restarting download")
    else:
        // Fresh download — clear any stale progress
        ota_progress_clear()
    
    // Download with per-chunk hash verification and checkpoint journaling.
    // Checkpoints written every PROGRESS_CHECKPOINT_INTERVAL bytes (32 KB).
    let download_handle = http_download_begin(info.download_url, start_offset = bytes_written)
    let last_checkpoint = bytes_written
    
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
                
                // Write progress checkpoint every 32 KB (or at download end)
                if bytes_written - last_checkpoint >= PROGRESS_CHECKPOINT_INTERVAL
                   or bytes_written == info.image_size:
                    let truncated_hash = hash_clone(running_hash).finalize()[0..8]
                    ota_progress_write(bytes_written, truncated_hash)
                    last_checkpoint = bytes_written
                
            case End:
                break
            case Error(e):
                // Preserve progress for resume — don't clear journal
                http_download_abort(download_handle)
                return Err(ERR-OTA-DOWNLOAD-001)
    
    let final_hash = hash_finalize(running_hash)
    
    // Verify full image hash
    assert(constant_time_compare(final_hash, info.image_hash, 32), ERR-OTA-DOWNLOAD-002)
    assert(bytes_written == info.image_size, ERR-OTA-DOWNLOAD-002)
    
    // Download complete — clear progress journal
    ota_progress_clear()
    
    // ── Phase 4: Verify Downloaded Image ──
    // OTA-layer verification of the OTA Image Package.
    // Full cryptographic verification (ECDH decrypt + Ed25519 + SHA-256)
    // is deferred to the bootloader's Phase 4.5-6 (defense-in-depth:
    // the bootloader independently re-verifies everything).
    state = OTA_VERIFY
    
    // Read back and verify OTA package structure
    let image_data = storage_read_slot(inactive_slot)
    let ota_header = parse_ota_header(image_data)
    assert(ota_header.is_ok(), ERR-BOOT-PARSE-011)
    assert(ota_header.unwrap().magic == 0x534F5441, ERR-BOOT-PARSE-011)
    assert(ota_header.unwrap().payload_length > 0, ERR-BOOT-PARSE-004)
    
    // Verify download integrity: SHA-256 over the full OTA package
    let computed_hash = crypto_compute_hash(image_data)
    assert(constant_time_compare(computed_hash, info.image_hash, 32), ERR-OTA-DOWNLOAD-002)
    
    // Set FLAG_OTA_PENDING so the bootloader knows this is an encrypted OTA image
    // The bootloader will perform ECDH + GCM decrypt + Ed25519 verify + KD_Storage re-encrypt
    let header = parse_header(image_data)  // ImageHeader is inside OTAHeader — may not be parseable yet
    // FLAG_OTA_PENDING is set by the OTA packaging process in the ImageHeader inside the
    // encrypted payload. The OTA layer just stores the package; the flag is already set.
    
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
| DOWNLOAD → VERIFY | downloaded_bytes == image_size AND SHA-256 over OTA package matches |
| VERIFY → INSTALL | OTA package structure valid (magic, payload_length) + SHA-256 matches |
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
| ERR-OTA-DOWNLOAD-001 | Resume from last progress journal checkpoint (HTTP Range, §9) |
| ERR-OTA-DOWNLOAD-002 | Discard; restart download from beginning |
| ERR-OTA-DOWNLOAD-003 | Abort; wait for storage availability or battery charge |
| ERR-OTA-INSTALL-001/002 | Retry write (max 3 attempts); then mark slot bad; try other slot |
| ERR-OTA-ACTIVATE-001/002 | Automatic rollback to fallback slot |

---

## 9. OTA Progress Journal API

The progress journal enables download resume after power loss or network interruption. It stores checkpoints in a ring buffer at NOR 0x20_3000 (4 KB sector, 64 records × 64 bytes). Each checkpoint records the bytes written and a truncated running hash for integrity verification on resume.

### 9.1 ProgressRecord Structure

```
struct ProgressRecord:                                // 64 bytes total
    magic:          u32 = 0x4F544150 ("OTAP")        // [0]  Record magic
    sequence:       u32                               // [4]  Chunk counter (0, 1, 2, ...)
    target_slot:    u8                                // [8]  SLOT_A or SLOT_B
    flags:          u8                                // [9]  Reserved (must be 0)
    reserved:       u8[2]                             // [10] Reserved
    bytes_written:  u32                               // [12] Total bytes written so far
    total_size:     u32                               // [16] Expected total image size
    expected_hash:  [u8; 32]                          // [20] SHA-256 of full image (from UpdateInfo)
    running_hash:   [u8; 8]                           // [52] Truncated SHA-256 of data written so far
    crc32:          u32                               // [60] CRC-32 over bytes 0..59
```

### 9.2 Progress Journal Operations

```
const PROGRESS_JOURNAL_BASE:   u32 = 0x20_3000
const PROGRESS_RECORD_SIZE:    u32 = 64
const PROGRESS_SECTOR_SIZE:    u32 = 4096
const PROGRESS_MAX_RECORDS:    u32 = 64
const PROGRESS_CHECKPOINT_INTERVAL: u32 = 32768  // 32 KB between checkpoints


/// Initialize a new progress journal entry for a download.
/// Called once at the start of Phase 3 (DOWNLOAD).
function ota_progress_init(slot: SlotID, total_size: u32, expected_hash: &[u8; 32]):
    // Erase the progress journal sector if it's been used before
    // (only if first record slot is occupied — lazy erase)
    let first_word = nor_spi_read(PROGRESS_JOURNAL_BASE, 4)
    if first_word != 0xFFFFFFFF:
        ota_progress_clear()


/// Write a progress checkpoint to the journal ring buffer.
/// Called every PROGRESS_CHECKPOINT_INTERVAL bytes during download.
/// No sector erase unless the ring wraps (once per 64 checkpoints).
function ota_progress_write(bytes_written: u32, running_hash: &[u8; 8]):
    let last = ota_progress_read_last()
    let new_seq = (last.valid) ? last.sequence + 1 : 0

    let write_off = ota_progress_find_next_free()
    if write_off >= PROGRESS_SECTOR_SIZE:
        // Ring full — erase and restart
        nor_spi_write_enable()
        nor_spi_erase_sector(PROGRESS_JOURNAL_BASE)
        nor_spi_wait_busy()
        write_off = 0

    let mut record = [0u8; 64]
    record[0..4]   = 0x4F544150 as u32          // "OTAP"
    record[4..8]   = new_seq as u32
    record[8]      = g_progress.target_slot as u8
    record[9]      = 0x00                        // flags
    record[10..12] = [0u8, 0u8]                  // reserved
    record[12..16] = bytes_written as u32
    record[16..20] = g_progress.total_size as u32
    record[20..52] = g_progress.expected_hash
    record[52..60] = running_hash[0..8]
    record[60..64] = crc32(record[0..60])

    nor_spi_page_program(PROGRESS_JOURNAL_BASE + write_off, record, 64)

    // Verify write
    let verify = nor_spi_read(PROGRESS_JOURNAL_BASE + write_off, 64)
    assert(memcmp(record, verify, 64) == 0, ERR-STOR-WRITE-001)


/// Read the most recent valid progress record.
/// Scans backward from end of sector for first record with valid magic + CRC-32.
/// Returns None if sector is empty or all records are corrupt.
function ota_progress_read_last() -> Option<ProgressRecord>:
    for off in (PROGRESS_SECTOR_SIZE - PROGRESS_RECORD_SIZE) down to 0
             step PROGRESS_RECORD_SIZE:
        let data = nor_spi_read(PROGRESS_JOURNAL_BASE + off, 64)
        if data[0..4] != 0xFFFFFFFF:                    // Not erased
            if data[0..4] as u32 == 0x4F544150           // Magic matches
               and crc32(data[0..60]) == data[60..64] as u32:  // CRC valid
                return Some(ProgressRecord{
                    magic:         0x4F544150,
                    sequence:      data[4..8] as u32,
                    target_slot:   data[8] as SlotID,
                    flags:         data[9],
                    bytes_written: data[12..16] as u32,
                    total_size:    data[16..20] as u32,
                    expected_hash: data[20..52],
                    running_hash:  data[52..60],
                })
    return None


/// Find the next free record slot (first erased 64-byte aligned offset).
function ota_progress_find_next_free() -> u32:
    for off in 0..PROGRESS_SECTOR_SIZE step PROGRESS_RECORD_SIZE:
        let header = nor_spi_read(PROGRESS_JOURNAL_BASE + off, 4)
        if header == 0xFFFFFFFF:
            return off
    return PROGRESS_SECTOR_SIZE  // Sector full


/// Clear the progress journal (erase sector). Called after successful download.
function ota_progress_clear():
    nor_spi_write_enable()
    nor_spi_erase_sector(PROGRESS_JOURNAL_BASE)
    nor_spi_wait_busy()


// Global progress state (volatile, rebuilt from journal on resume)
static g_progress: struct {
    target_slot:    SlotID,
    total_size:     u32,
    expected_hash:  [u8; 32],
}
```

### 9.3 Resume Flow

```
// Called at ota_perform_update() entry, before starting download.
// Determines whether to resume or restart.

function ota_try_resume(inactive_slot: SlotID,
                        info: UpdateInfo) -> (u32, HashState):
    let prev = ota_progress_read_last()

    if not prev.valid:
        return (0, hash_init(SHA256))  // Fresh download

    // Validate resume is for the same update
    if prev.target_slot != inactive_slot:
        ota_progress_clear()
        return (0, hash_init(SHA256))

    if prev.expected_hash != info.image_hash:
        ota_progress_clear()
        return (0, hash_init(SHA256))

    // Verify existing data integrity
    let existing = storage_read_slot_range(inactive_slot, 0, prev.bytes_written)
    let existing_hash = crypto_compute_hash(existing)

    if constant_time_compare(existing_hash[0..8], prev.running_hash, 8):
        // Data intact — rebuild running hash and resume
        let mut hash_state = hash_init(SHA256)
        hash_state = hash_update(hash_state, existing)
        return (prev.bytes_written, hash_state)
    else:
        // Data corrupt — must restart
        storage_erase_slot(inactive_slot)?
        ota_progress_clear()
        log("OTA resume: existing data corrupt, restarting")
        return (0, hash_init(SHA256))
```

### 9.4 Power-Loss Analysis

| Failure Point | What Happens | Resume Behavior |
|---------------|-------------|-----------------|
| Power loss between checkpoints | Last checkpoint ≤ 32 KB behind actual write position | Resume from last checkpoint. HTTP Range re-downloads up to 32 KB of duplicate data — harmless. |
| Power loss during checkpoint write | Partial 64 B record, CRC-32 mismatch | `read_last()` skips corrupt record, finds previous valid checkpoint |
| Power loss during sector erase (ring wrap) | All records lost | Fresh download required — no worse than no progress journal |
| Power loss during slot write (not checkpoint) | NOR page program incomplete | Slot data at that offset is 0xFF. Running hash mismatch on resume → restart. |
| Backend changes image between attempts | expected_hash differs from stored | `ota_try_resume()` detects hash mismatch, clears journal, fresh download |
