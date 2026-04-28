# Boot Flow Pseudocode Specification

**Document ID:** SUB-BOOT-FLOW-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document specifies the concrete boot algorithm in pseudocode form. It defines the exact verification sequence, state transitions, guard conditions, and error handling that a conforming SBOP boot implementation must follow.

---

## 2. Main Boot Flow

```
function sbop_boot() -> Never:
    // Capture original reset cause before platform_init() clears RCC->CSR.
    // HAL_Init() reads and clears RCC->CSR flags — must capture first.
    // Stored in RTC->BKP3R (backup domain) so the application can read the
    // real reset reason (POR, IWDG, NRST pin, BOR, etc.) instead of Pass 1's
    // SFTRSTF. BKP3R survives system reset — no NOR flash write needed.
    let original_reset_cause = RCC->CSR

    // ── Phase 1: Hardware Initialization (INIT) ──
    state = INIT
    assert_eq(platform_init(), OK)

    // Preserve reset cause in backup domain.
    // BKP3R survives system reset — application reads RTC->BKP3R directly.
    RTC->BKP3R = original_reset_cause

    // Check if this is a post-IWDG intentional reset (two-pass boot, §3.13)
    // RTC BKP0R survives system reset (backup domain); lost on POR/BOR
    if RTC->BKP0R == IWDG_RESET_MAGIC:   // 0x49574447 ("IWGD")
        // Pass 2: IWDG is in hardware reset state (stopped by the reset)

        // VBAT+POR edge case: if VBAT powers the backup domain (coin cell),
        // RTC registers survive VDD power cycles but AXI SRAM does not.
        // BKP1R holds CRC-32 of AXI SRAM first 4 KB (set in Pass 1 EXECUTE).
        // If CRC mismatches, a POR occurred while RTC was alive — fall through
        // to full Pass 1 verification.
        let crc_now = crc32_compute(AXI_SRAM_BASE, AXI_SRAM_INTEGRITY_LEN)
        if crc_now != RTC->BKP1R:
            RTC->BKP0R = 0
            RTC->BKP1R = 0
            // Falls through to Pass 1 full verification below
        else:
            // AXI SRAM intact — safe to jump directly to application
            RTC->BKP0R = 0  // Clear magic for next cold boot
            jump_to_prepared_application()  // Never returns (~5 ms path)
            // Application payload is already in AXI SRAM from Pass 1

    // ── Pass 1: Full boot with IWDG protection ──

    // Initialize IWDG (independent hardware watchdog, LSI ~32 kHz, 800 ms timeout)
    // Must be done before any external input to prevent hang-induced deadlock
    iwdg_init(prescaler=32, reload=820)   // 800 ms timeout
    IWDG_REFRESH()

    // Verify NOR flash presence via JEDEC ID (9FH command)
    let jedec_id = nor_spi_read_jedec_id()
    // Supported: Boya BY25Q32ES (0x68) and Winbond W25Q32JV (0xEF)
    assert(jedec_id[0] == 0x68 or jedec_id[0] == 0xEF, ERR-HW-NOR-001)
    assert(jedec_id[1] == 0x40, ERR-HW-NOR-001)  // SPI NOR flash
    assert(jedec_id[2] == 0x16, ERR-HW-NOR-001)  // 4 MB (32 Mbit)

    // Verify NOR Block Protect bits BEFORE any metadata read.
    // Metadata at 0x20_0000+ must be write-protected before we trust it.
    IWDG_REFRESH()
    let sr1 = nor_spi_read_status_register()
    if (sr1 & BP2_BIT) == 0:
        // BP2 not set — metadata area is unprotected. Re-apply immediately.
        tamper_log_write(TAMPER_BP_BITS_CLEARED)
        nor_spi_write_enable()
        nor_spi_write_status_register(BP2_BIT | SRP1_ENABLE)
        let sr1_v2 = nor_spi_read_status_register()
        if (sr1_v2 & BP2_BIT) == 0:
            return Err(ERR-HW-NOR-002)  // NOR not accepting BP commands

    // Check tamper state before anything else
    let tamper_state = tamper_detect_get_state()
    if tamper_state == TAMPERED or tamper_state == COMPROMISED:
        handle_tamper_failure(tamper_state)
        // Does not return

    // ── Phase 2: Select Boot Slot ──
    state = SELECT_SLOT
    IWDG_REFRESH()
    let boot_slot = storage_get_boot_slot()
    assert(boot_slot == SLOT_A or boot_slot == SLOT_B, ERR-BOOT-STATE-001)

    // ── Phase 3-9: Attempt to verify and boot the selected slot ──
    let result = boot_verify_and_execute_slot(boot_slot)

    if result == OK:
        // Never returns — jumped to application
        unreachable

    // ── Selected slot failed — try the fallback slot ──
    // IWDG is active (Pass 1, 800 ms timeout). Each NOR write below
    // may take 10-45 ms. Refresh before and after the fallback sequence.
    IWDG_REFRESH()
    error_log_write(state, result.error)
    IWDG_REFRESH()
    storage_set_slot_status(boot_slot, INVALID)

    let fallback_slot = (boot_slot == SLOT_A) ? SLOT_B : SLOT_A
    let fallback_status = storage_get_slot_status(fallback_slot)

    if fallback_status == ACTIVE or fallback_status == FALLBACK or fallback_status == VERIFIED:
        // Fallback slot has a valid image — try it.
        // VERIFIED means Gate 1 passed (OTA download + backend sig OK),
        // Gate 2 hasn't run yet — still a valid fallback candidate.
        state = SELECT_SLOT
        IWDG_REFRESH()
        storage_set_boot_slot(fallback_slot)
        IWDG_REFRESH()
        metadata_set_boot_attempt_count(0)
        metadata_set_prev_boot_phase(0x00)

        let fallback_result = boot_verify_and_execute_slot(fallback_slot)
        if fallback_result == OK:
            // Never returns — jumped to fallback application
            unreachable

        // Both slots failed — enter FAILSAFE
        IWDG_REFRESH()
        storage_set_slot_status(fallback_slot, INVALID)

    // ── Both slots dead — enter FAILSAFE ──
    boot_enter_failsafe(ERR-BOOT-STATE-004)  // No bootable slot
```

**`jump_to_prepared_application()` — Pass 2 fast path (~5 ms):**

```
// Called on Pass 2 when RTC->BKP0R == IWDG_RESET_MAGIC.
// The application payload is already decrypted and verified in AXI SRAM
// from Pass 1 (AXI SRAM survives system reset, lost only on POR/BOR).
// IWDG is in hardware reset state (disabled). No verification needed.
function jump_to_prepared_application() -> Never:
    // RTC->BKP3R already holds the original RCC->CSR from Pass 1 INIT.
    // Application can read RTC->BKP3R to determine the real reset cause.
    // Application binary starts with vector table at AXI_SRAM_BASE
    let vt = (uint32_t *)AXI_SRAM_BASE

    let initial_sp = vt[0]   // MSP = AXI_SRAM[0]
    let reset_pc   = vt[1]   // PC  = AXI_SRAM[4]

    assert(initial_sp >= AXI_SRAM_BASE, ERR-BOOT-PARSE-007)
    assert(initial_sp < AXI_SRAM_BASE + AXI_SRAM_SIZE, ERR-BOOT-PARSE-007)
    assert(reset_pc >= AXI_SRAM_BASE, ERR-BOOT-PARSE-007)
    assert(reset_pc < AXI_SRAM_BASE + AXI_SRAM_SIZE, ERR-BOOT-PARSE-007)

    // Set VTOR to application's vector table in AXI SRAM
    SCB->VTOR = AXI_SRAM_BASE

    // Clear bootloader state from DTCM + SRAM1/2
    secure_zeroize_boot_workspace()

    // Jump to application — never returns
    // (inline assembly: msr msp, initial_sp; bx reset_pc)
    platform_jump_to_firmware(initial_sp, reset_pc)
```

**`boot_verify_and_execute_slot()` — single-slot verification (Phases 3-11):**

```
function boot_verify_and_execute_slot(slot: SlotID) -> Result<Never, BootError>:

    // ── Phase 3: Load Image ──
    state = LOAD_IMAGE
    IWDG_REFRESH()
    let image_data = storage_read_slot(slot)
    if image_data == null: return Err(ERR-STOR-READ-001)
    if image_data.len < MIN_IMAGE_SIZE: return Err(ERR-BOOT-PARSE-001)

    // ── Phase 4: Parse Header ── (all FIH-protected checks)
    state = PARSE_HEADER
    IWDG_REFRESH()
    let header = parse_header(image_data)
    if header == null: return Err(ERR-BOOT-PARSE-001)
    if header.magic != 0x53424F50: return Err(ERR-BOOT-PARSE-001)
    if header.header_version > MAX_SUPPORTED_VERSION: return Err(ERR-BOOT-PARSE-002)
    if header.header_size < 80: return Err(ERR-BOOT-PARSE-003)
    if header.image_size == 0 or header.image_size > MAX_IMAGE_SIZE:
        return Err(ERR-BOOT-PARSE-004)
    if !header.reserved_is_zero(): return Err(ERR-BOOT-PARSE-005)

    // HMAC-SHA-256-128(KD, header[0..56]) — SPI bus integrity (§3.1 R-IMG-008)
    let header_hmac_input = image_data[0..56]
    let expected_hmac = crypto_hmac_sha256(KD, header_hmac_input)[0..16]
    if constant_time_compare(header.header_hmac, expected_hmac, 16) != true:
        return Err(ERR-BOOT-PARSE-010)

    let payload = image_data[header.header_size .. header.header_size + header.image_size]
    let sig_block = parse_signature_block(image_data, header)
    if sig_block == null: return Err(ERR-BOOT-PARSE-006)

    // ── Phase 5: Verify Signature ──
    state = VERIFY_SIGNATURE
    IWDG_REFRESH()

    // Parse signature block fields (key_index, public_key, signature, crc32)
    // SignatureBlock CRC-32 validated in Phase 4 (parse_signature_block)
    let key_index  = sig_block.key_index
    let public_key = sig_block.public_key
    let signature  = sig_block.signature

    // Key fingerprint verification (NXP CSM-style multi-key, §3.10.9)
    // Only for multi-key mode (key_index 0-3). Legacy mode (0xFF) uses baked-in key.
    if key_index != 0xFF:
        // Verify this key slot has NOT been revoked
        let revocation = nor_spi_read_sr1_u16(KEY_REVOCATION_OFFSET)
        if ((revocation >> key_index) & 1) == 0:
            return Err(ERR-BOOT-CRYPTO-004)  // Key slot revoked

        // Verify public key fingerprint matches OTP
        let expected_fp = nor_spi_read_sr1(KEY_FINGERPRINT_BASE + key_index * 32, 32)
        let actual_fp   = crypto_compute_hash(public_key, SHA256)
        if constant_time_compare(actual_fp, expected_fp, 32) != true:
            return Err(ERR-BOOT-CRYPTO-004)  // Fingerprint mismatch
    else:
        // Legacy single-key mode: use baked-in key from bootloader .rodata
        public_key = LEGACY_KI_PUBLIC  // PC-ROP protected

    // Refreshed per 4 KB during streaming decrypt (see platform eval doc §3.13)

    let sig_data = concat(image_data[0 .. header.header_size], payload)
    let sig_valid = ed25519_verify(sig_data, signature, public_key)
    if sig_valid != true: return Err(ERR-BOOT-CRYPTO-001)
    // FIH redundant verify (fault injection defense)
    let sig_valid_v2 = ed25519_verify(sig_data, signature, public_key)
    if sig_valid_v2 != true: return Err(ERR-BOOT-CRYPTO-001)
    IWDG_REFRESH()

    // ── Phase 6: Verify Integrity ──
    state = VERIFY_INTEGRITY
    let computed_hash = crypto_compute_hash(payload)         // HW SHA-256 (primary)
    let computed_hash_sw = crypto_compute_hash_sw(payload)   // Software SHA-256 cross-check
    // Both paths must agree — detects HW HASH glitch or FIH attack
    if constant_time_compare(computed_hash, computed_hash_sw, 32) != true:
        return Err(ERR-BOOT-CRYPTO-005)
    let hash_match = constant_time_compare(computed_hash, header.hash, 32)
    if hash_match != true: return Err(ERR-BOOT-CRYPTO-002)
    // FIH redundant compare
    let hash_match_v2 = constant_time_compare(computed_hash, header.hash, 32)
    if hash_match_v2 != true: return Err(ERR-BOOT-CRYPTO-002)
    IWDG_REFRESH()

    // ── Phase 7: Check Version ──
    state = CHECK_VERSION
    IWDG_REFRESH()
    let stored_version = storage_read_version_counter()
    if stored_version == VERSION_ERR: return Err(ERR-BOOT-VERSION-002)
    if header.firmware_version < stored_version: return Err(ERR-BOOT-VERSION-001)
    if header.firmware_version == 0: return Err(ERR-BOOT-VERSION-003)

    // Redundant version check (fault injection defense)
    let stored_version_v2 = storage_read_version_counter()
    if stored_version != stored_version_v2: return Err(ERR-BOOT-STATE-003)
    if header.firmware_version < stored_version_v2: return Err(ERR-BOOT-VERSION-001)

    // ── Phase 8: Commit Version ──
    state = COMMIT_VERSION
    IWDG_REFRESH()
    if header.firmware_version > stored_version:
        let write_ok = storage_write_version_counter(header.firmware_version)
        if write_ok != true: return Err(ERR-STOR-WRITE-001)

    // ── Phase 9: Mark Slot Active ──
    state = MARK_ACTIVE
    IWDG_REFRESH()
    let mark_ok = storage_set_slot_status(slot, ACTIVE)
    if mark_ok != true: return Err(ERR-STOR-WRITE-001)

    // ── Phase 10: Lock Boot ──
    state = LOCK_BOOT
    IWDG_REFRESH()
    boot_lock()  // Disable boot-time modifications; protect Zone 1 memory

    // ── Phase 11: Execute ──
    state = EXECUTE
    IWDG_REFRESH()
    let entry_point = payload.entry_point()
    if entry_point == null: return Err(ERR-BOOT-PARSE-007)

    // Mark that we reached EXECUTE (for boot health tracking)
    metadata_set_prev_boot_phase(0x01)

    // Update boot metrics
    boot_metrics_increment()

    // RTC->BKP3R already holds the original RCC->CSR value (captured before
    // HAL_Init cleared it in Pass 1 INIT). It survives this system reset, so
    // the application can read RTC->BKP3R to determine the real reset cause
    // (POR, BOR, IWDG, NRST pin, etc.) instead of seeing Pass 1's SFTRSTF.
    //
    // Compute CRC-32 over AXI SRAM first 4 KB for Pass 2 integrity check.
    // If VBAT powers RTC domain, RTC registers survive POR but AXI SRAM does not.
    // Pass 2 verifies BKP1R against current AXI SRAM content before jumping.
    RTC->BKP1R = crc32_compute(AXI_SRAM_BASE, AXI_SRAM_INTEGRITY_LEN)
    //
    // Stop IWDG before jumping to application:
    //   IWDG cannot be disabled by software (RM0433). The only way to stop
    //   it is a system reset. Set RTC backup register magic value so Pass 2
    //   (next boot) knows this was intentional and skips IWDG init.
    RTC->BKP0R = IWDG_RESET_MAGIC   // 0x49574447
    NVIC_SystemReset()              // System reset → IWDG stops → sbop_boot() Pass 2
    // Does not return — Pass 2 jumps directly to application
```

---

## 3. Fail-Safe Handler

```
function handle_boot_error(error: BootError) -> Never:
    // Enter FAILSAFE state
    state = FAILSAFE

    // Log the error (if possible without executing firmware)
    error_log_write(state, error)

    // Zeroize CRYP hardware key registers
    // CRYP registers survive soft reset — must be explicitly cleared
    crypto_hw_key_zeroize()
    
    // Signal fail-safe to external monitor
    platform_signal_failsafe(error.code)
    
    // Zeroize any sensitive data in RAM
    secure_zeroize_boot_workspace()
    
    // Enter infinite wait for external recovery.
    // IWDG is still active — must refresh to prevent 800 ms reset.
    loop:
        IWDG_REFRESH()
        
        // Optionally: listen for recovery command on minimal interface
        if recovery_interface_data_available():
            let cmd = recovery_interface_read()
            if cmd.type == RECOVERY_OTA_INITIATE:
                IWDG_REFRESH()
                enter_recovery_boot()
        
        platform_idle()  // Low-power wait
```

---

## 4. Recovery Boot

```
function enter_recovery_boot():
    // Minimal boot for OTA recovery only
    // Does NOT execute application firmware
    
    let recovery_slot = storage_get_fallback_slot()
    if recovery_slot == SLOT_INVALID:
        // No valid fallback; wait for full OTA
        return
    
    // Attempt to boot the fallback slot
    // (Reuses verification functions from main boot)
    
    let image_data = storage_read_slot(recovery_slot)
    if verify_image_minimal(image_data) == OK:
        storage_set_boot_slot(recovery_slot)
        system_reset()  // Full reboot into recovery slot
    else:
        // Recovery failed; wait for external intervention
        platform_signal_failsafe(ERR-BOOT-STATE-001)
```

---

## 5. Minimal Image Verification

```
function verify_image_minimal(image_data: &[u8]) -> Result<(), BootError>:
    // Fast-path verification for recovery boot
    // Only checks that matter for determining bootability
    
    let header = parse_header(image_data)?
    
    // Verify signature (non-negotiable)
    let payload = image_data[header.header_size .. header.header_size + header.image_size]
    let sig_block = parse_signature_block(image_data, header)?
    // Multi-key CSM — verify fingerprint + revocation + signature using embedded public key
    if sig_block.key_index != 0xFF:
        let revocation = nor_spi_read_sr1_u16(KEY_REVOCATION_OFFSET)
        if ((revocation >> sig_block.key_index) & 1) == 0:
            return Err(ERR-BOOT-CRYPTO-004)
        let expected_fp = nor_spi_read_sr1(KEY_FINGERPRINT_BASE + sig_block.key_index * 32, 32)
        let actual_fp = crypto_compute_hash(sig_block.public_key, SHA256)
        ensure(constant_time_compare(actual_fp, expected_fp, 32), ERR-BOOT-CRYPTO-004)
        let verify_key = sig_block.public_key
    else:
        let verify_key = LEGACY_KI_PUBLIC  // Baked-in .rodata key
    let sig_valid = ed25519_verify(
        concat(image_data[0..header.header_size], payload),
        sig_block.signature,
        verify_key
    )
    ensure(sig_valid, ERR-BOOT-CRYPTO-001)
    
    // Verify hash
    let hash = crypto_compute_hash(payload)
    ensure(constant_time_compare(hash, header.hash, 32), ERR-BOOT-CRYPTO-002)
    
    // Check version (allow rollback for recovery)
    // Recovery may boot older firmware to re-establish OTA capability
    ensure(header.firmware_version > 0, ERR-BOOT-VERSION-003)
    
    return OK
```

---

## 6. Tamper Failure Handler

```
function handle_tamper_failure(tamper_state: TamperState) -> Never:
    state = TAMPER_FAILSAFE
    
    // Log tamper event
    tamper_log_write(tamper_state)
    
    // Zeroize device key (KD)
    // Root key (KR) is presumably in hardware-protected storage
    key_zeroize(DEVICE)
    key_zeroize(DEBUG_AUTH)
    
    // Signal tamper to external monitor
    platform_signal_tamper(tamper_state)
    
    // If tamper is critical, attempt permanent lock
    if tamper_state == COMPROMISED:
        debug_lock_permanent()
    
    // Enter infinite wait
    loop:
        platform_idle()
```

---

## 7. State Transition Table (Boot)

| Current State | Condition | Next State | Guard Check |
| --- | --- | --- | --- |
| INIT (Pass 1) | RTC->BKP0R == 0 (cold boot) + platform_init() OK | SELECT_SLOT | Tamper state NORMAL |
| INIT (Pass 2) | RTC->BKP0R == IWDG_RESET_MAGIC | JUMP_TO_APP | Application already in AXI SRAM from Pass 1 |
| INIT (Pass 1) | tamper_state != NORMAL | TAMPER_FAILSAFE | — |
| SELECT_SLOT | boot_slot valid | LOAD_IMAGE | Slot in {SLOT_A, SLOT_B} |
| SELECT_SLOT | boot_slot invalid | FAILSAFE | ERR-BOOT-STATE-001 |
| LOAD_IMAGE | image read OK | PARSE_HEADER | image_data != null |
| LOAD_IMAGE | read error | FAILSAFE | ERR-STOR-READ-001 |
| PARSE_HEADER | all fields valid | VERIFY_SIGNATURE | Section 2 rules R-IMG-001..006 |
| PARSE_HEADER | any field invalid | FAILSAFE | ERR-BOOT-PARSE-001..006 |
| VERIFY_SIGNATURE | signature valid | VERIFY_INTEGRITY | crypto_verify_signature == true |
| VERIFY_SIGNATURE | signature invalid | FAILSAFE | ERR-BOOT-CRYPTO-001 |
| VERIFY_INTEGRITY | hash match | CHECK_VERSION | constant_time_compare == true |
| VERIFY_INTEGRITY | hash mismatch | FAILSAFE | ERR-BOOT-CRYPTO-002 |
| CHECK_VERSION | version >= stored | COMMIT_VERSION | firmware_version >= stored_version |
| CHECK_VERSION | rollback detected | FAILSAFE | ERR-BOOT-VERSION-001 |
| COMMIT_VERSION | write OK | MARK_ACTIVE | storage_write returns OK |
| COMMIT_VERSION | write error | FAILSAFE | ERR-STOR-WRITE-001 |
| MARK_ACTIVE | mark OK | LOCK_BOOT | slot status set to ACTIVE |
| MARK_ACTIVE | mark error | FAILSAFE | ERR-STOR-WRITE-001 |
| LOCK_BOOT | lock OK | EXECUTE | Zone 1 memory protected |
| EXECUTE (Pass 1) | RTC magic written + NVIC_SystemReset | Pass 2 INIT | IWDG stops on reset; AXI SRAM preserved |
| PASS_2_INIT | RTC magic detected | JUMP_TO_APP | Application already in AXI SRAM |
| JUMP_TO_APP | Jump to AXI SRAM application | — (application runs) | VTOR + MSP + PC valid |

---

## 8. Reset Cause Preservation

The two-pass boot (Pass 1 → NVIC_SystemReset → Pass 2) would normally cause the application to see only SFTRSTF (software reset) in RCC->CSR, losing the original reset cause. To preserve the real reset cause:

```
Pass 1 INIT:
    original_reset_cause = RCC->CSR    // Capture before HAL_Init() clears flags
    RTC->BKP3R = original_reset_cause  // Store in backup domain (survives reset)

Pass 1 EXECUTE:
    // RTC->BKP3R already holds the original value — no action needed here
    RTC->BKP0R = IWDG_RESET_MAGIC
    NVIC_SystemReset()                 // BKP3R survives this reset

Pass 2 INIT → jump_to_prepared_application():
    // RTC->BKP3R still holds the original RCC->CSR from Pass 1 INIT
    // Application can read RTC->BKP3R directly

Application:
    // Enable RTC clock, then read BKP3R
    RCC->BDCR |= RCC_BDCR_RTCEN;
    uint32_t cause = RTC->BKP3R;
    // cause == 0x00000000 → cold POR/BOR (not yet written)
    // cause has RCC_CSR_PORRSTF  → power-on reset
    // cause has RCC_CSR_PINRSTF  → NRST pin reset
    // cause has RCC_CSR_IWDGRSTF → IWDG watchdog reset
```

**Design rationale:** RTC backup registers are in the backup domain — they survive any system reset and are lost only on POR/BOR (if VBAT unpowered) or intentional RTC domain reset. This makes them the natural handoff mechanism for the reset cause: no NOR flash write needed, no KV entry consumed, zero latency, and the bootloader only writes to BKP3R once in Pass 1.

---

## 9. Security Invariants (Enforced in Pseudocode)

| Invariant | Enforced By | Description |
| --- | --- | --- |
| I1 | Phases 5→6→7 must all pass before Phase 10 | No shortcut to EXECUTE |
| I2 | assert() on every phase | Any failure terminates flow |
| I3 | Redundant version check in Phase 7 | Detects fault injection on version comparison |
| I4 | constant_time_compare in Phase 6 | Prevents timing oracle on hash comparison |
| I5 | tamper_detect_get_state() in INIT | Tamper checked before any firmware interaction |
| I6 | key_zeroize in handle_tamper_failure | Keys destroyed before infinite wait |

---

## 10. Timing Budgets

**Pass 1 (IWDG-protected, single slot verification):**

| Phase | Maximum Allowed Time | Notes |
| --- | --- | --- |
| INIT (incl. IWDG start + NOR detect) | ~6 ms | Platform init + JEDEC ID read |
| SELECT_SLOT | < 1 ms | Simple metadata read |
| LOAD_IMAGE | < 100 ms | Depends on image size and flash speed |
| PARSE_HEADER | < 1 ms | Fixed-size parse |
| VERIFY_SIGNATURE | < 50 ms | Ed25519 verify |
| VERIFY_INTEGRITY | < 100 ms | SHA-256 over payload |
| CHECK_VERSION | < 1 ms | Simple integer comparison |
| COMMIT_VERSION | < 10 ms | Flash write |
| MARK_ACTIVE | < 10 ms | Flash write |
| LOCK_BOOT | < 1 ms | Memory protection enable |
| EXECUTE (RTC magic + NVIC_SystemReset) | < 1 ms | IWDG stops on reset |

**Pass 2 (IWDG stopped, fast path to application):**

| Phase | Maximum Allowed Time | Notes |
| --- | --- | --- |
| INIT (detect RTC magic) | ~2 ms | Platform init (minimal) |
| Jump to prepared application | ~3 ms | VTOR set, workspace clear, MSP+PC load |

**Total boot budget target:** < 200 ms (Pass 1) + ~5 ms (Pass 2) ≈ < 205 ms from cold reset to firmware execution. Well under the 300 ms SBOP target.
