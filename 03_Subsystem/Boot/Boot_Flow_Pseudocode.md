# Boot Flow Pseudocode Specification

**Document ID:** SUB-BOOT-FLOW-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

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

    // Test/confirm check: if the selected slot is in TESTING state, the
    // application from the previous boot did NOT call boot_set_confirmed().
    // This means the image failed to prove itself healthy — immediate rollback,
    // no N-attempt retry. Matches MCUboot's test/confirm model.
    let boot_slot_status = storage_get_slot_status(boot_slot)
    if boot_slot_status == TESTING:
        // Application never confirmed — image is not healthy
        error_log_write(SELECT_SLOT, ERR-BOOT-STATE-005)  // Unconfirmed image
        storage_set_slot_status(boot_slot, INVALID)
        // Fall through to fallback logic below
    else if boot_slot_status != ACTIVE and boot_slot_status != FALLBACK
         and boot_slot_status != VERIFIED and boot_slot_status != PENDING:
        // Slot is in an unexpected state
        return Err(ERR-BOOT-STATE-001)

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
        // Frequent journal reset: erase ring buffer to start fresh for fallback slot.
        // Critical journal is untouched — slot state transition handled above.
        frequent_journal_erase()

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
    let mut ota_verified = false

    // ── Phase 4.5: OTA Decrypt + Re-encrypt (FLAG_OTA_PENDING) ──
    // When FLAG_OTA_PENDING is set, the slot contains an OTA Image Package
    // (OTAHeader || AES-256-GCM(Payload) || GCM_Tag || Ed25519_Signature).
    // The bootloader must: ECDH → derive K_s → GCM decrypt → verify signature
    // on plaintext → re-encrypt with KD_Storage. After this phase, the slot
    // contains a standard SBOP image (device-unique encrypted).
    if (header.flags & FLAG_OTA_PENDING) != 0:
        state = OTA_DECRYPT
        IWDG_REFRESH()

        // ── Step 1: Parse OTA Image Package header ──
        // The OTAHeader precedes the ImageHeader in the slot when FLAG_OTA_PENDING
        // is set. Layout: OTAHeader (44 B) || EncryptedPayload || GCM_Tag (16 B)
        //              || Ed25519_Signature (64 B)
        let ota_header = parse_ota_header(image_data)
        if ota_header == null: return Err(ERR-BOOT-PARSE-011)
        if ota_header.magic != 0x534F5441: return Err(ERR-BOOT-PARSE-011)

        // ── Step 2: X25519 ECDH key agreement ──
        // Key agreement design (single OTA image for all devices):
        //
        //   All devices share the same X25519 key pair (KO).
        //   KO_private is in secure element (same for all, per-device unique
        //   is unnecessary since OTA encryption is defense-in-depth).
        //   KO_public is registered with the Dev Team.
        //
        //   Packaging (Dev Team):        Device (boot):
        //     eph_keypair = X25519_KeyGen()
        //     SharedSecret = X25519(      SharedSecret = X25519(
        //       eph_private,                KO_private,
        //       KO_public)                   eph_public)
        //
        //   By ECDH symmetry, both derive the same SharedSecret.
        //   Ephemeral public key (eph_public) is in OTA header.
        //
        //   Security: if KO_private is extracted from one device, all
        //   devices' OTA encryption is compromised. Mitigation: OTA encryption
        //   is defense-in-depth (TLS 1.3 is primary transport security). The
        //   real security boundary is the Ed25519 firmware signature (KI).

        // Read KO private key from secure element
        let ko_private = key_material_load(KEY_REF_KO)

        // Compute ECDH shared secret
        let shared_secret = x25519_ecdh(ko_private, ota_header.ephemeral_pubkey)
        if shared_secret == null: return Err(ERR-BOOT-CRYPTO-008)

        // ── Step 3: Derive K_s and GCM nonce via HKDF ──
        let fw_version_bytes = ota_header.firmware_version as [u8; 4]
        let info = "SBOP-OTA-v1" || fw_version_bytes
        let k_s = crypto_derive_key(shared_secret, salt="", info=info, output_len=32)
        if k_s.is_err(): return Err(ERR-BOOT-CRYPTO-008)

        let nonce_info = "SBOP-OTA-NONCE-v1"
        let gcm_nonce = crypto_derive_key(shared_secret, salt="", info=nonce_info, output_len=12)
        if gcm_nonce.is_err(): return Err(ERR-BOOT-CRYPTO-008)

        // ── Step 4: AES-256-GCM decrypt the encrypted payload ──
        let encrypted_payload = image_data[OTA_HEADER_SIZE + ota_header.aad_length ..
                                           OTA_HEADER_SIZE + ota_header.aad_length + ota_header.payload_length]
        let gcm_tag = image_data[OTA_HEADER_SIZE + ota_header.aad_length + ota_header.payload_length ..
                                 OTA_HEADER_SIZE + ota_header.aad_length + ota_header.payload_length + 16]
        let aad = image_data[OTA_HEADER_SIZE .. OTA_HEADER_SIZE + ota_header.aad_length]

        IWDG_REFRESH()
        let plaintext = aes_256_gcm_decrypt(
            key = k_s.unwrap(),
            nonce = gcm_nonce.unwrap(),
            aad = aad,
            ciphertext = encrypted_payload,
            tag = gcm_tag
        )
        if plaintext.is_err(): return Err(ERR-BOOT-CRYPTO-008)  // GCM auth failure

        // ── Step 5: Verify Ed25519 signature on plaintext ──
        // The OTA Image Package's trailing Ed25519 signature covers the
        // plaintext firmware (sign-then-encrypt ordering).
        let ota_signature = image_data[OTA_HEADER_SIZE + ota_header.aad_length +
                                        ota_header.payload_length + 16 ..
                                        OTA_HEADER_SIZE + ota_header.aad_length +
                                        ota_header.payload_length + 16 + 64]

        // Parse the decrypted SBOP image: ImageHeader || Firmware || (SignatureBlock)
        let decrypted_header = parse_header(plaintext)
        if decrypted_header == null: return Err(ERR-BOOT-PARSE-001)
        let decrypted_payload = plaintext[decrypted_header.header_size ..
                                           decrypted_header.header_size + decrypted_header.image_size]

        // Re-verify header HMAC over decrypted header
        let decrypted_hmac_input = plaintext[0..56]
        let decrypted_expected_hmac = crypto_hmac_sha256(KD, decrypted_hmac_input)[0..16]
        if constant_time_compare(decrypted_header.header_hmac, decrypted_expected_hmac, 16) != true:
            return Err(ERR-BOOT-PARSE-010)

        // FIH: Verify the OTA Ed25519 signature on plaintext (header || firmware)
        let ota_sig_data = plaintext[0 .. decrypted_header.header_size + decrypted_header.image_size]
        let ota_sig_valid = ed25519_verify(ota_sig_data, ota_signature, LEGACY_KI_PUBLIC)
        if ota_sig_valid != true: return Err(ERR-BOOT-CRYPTO-001)
        let ota_sig_valid_v2 = ed25519_verify(ota_sig_data, ota_signature, LEGACY_KI_PUBLIC)
        if ota_sig_valid_v2 != true: return Err(ERR-BOOT-CRYPTO-001)

        IWDG_REFRESH()

        // ── Step 6: Re-encrypt plaintext with KD_Storage (device-unique) ──
        // KD_Storage = HKDF(KD, "SBOP-STORAGE-v1") — unique per device
        let kd_storage = crypto_derive_key(KD, salt="", info="SBOP-STORAGE-v1", output_len=32)
        if kd_storage.is_err(): return Err(ERR-BOOT-CRYPTO-008)

        // Generate random nonce for KD_Storage encryption (stored in slot metadata)
        let storage_nonce = trng_random_bytes(12)
        let storage_nonce_write_ok = storage_write_iv(slot, storage_nonce)
        if storage_nonce_write_ok != true: return Err(ERR-STOR-WRITE-001)

        // AES-256-GCM encrypt the plaintext SBOP image
        // AAD binds the ciphertext to firmware version and slot
        let storage_aad = decrypted_header.firmware_version as [u8; 4] || (slot as u8)
        let re_encrypted = aes_256_gcm_encrypt(
            key = kd_storage.unwrap(),
            nonce = storage_nonce,
            aad = storage_aad,
            plaintext = plaintext
        )
        if re_encrypted.is_err(): return Err(ERR-BOOT-CRYPTO-008)

        let re_encrypted_payload = re_encrypted.ciphertext
        let re_encrypted_tag    = re_encrypted.tag

        IWDG_REFRESH()

        // ── Step 7: Write re-encrypted image to slot ──
        // Clear FLAG_OTA_PENDING — subsequent boots see a normal image
        decrypted_header.flags &= ~FLAG_OTA_PENDING
        decrypted_header.flags |= FLAG_ENCRYPTED  // Now device-unique encrypted

        // Recompute header HMAC
        let new_hmac_input = serialize_header_prefix(decrypted_header)[0..56]
        let new_header_hmac = crypto_hmac_sha256(KD, new_hmac_input)[0..16]
        decrypted_header.header_hmac = new_header_hmac

        // Write: ImageHeader (80 B) || EncryptedPayload || GCM_Tag (16 B) || SignatureBlock
        storage_erase_slot(slot)
        storage_write(slot, 0, serialize_header(decrypted_header))
        storage_write(slot, 80, re_encrypted_payload)
        storage_write(slot, 80 + re_encrypted_payload.len, re_encrypted_tag)

        // Construct SignatureBlock from the verified OTA signature
        // (OTA signature is just the 64-byte Ed25519 value; wrap in SignatureBlock)
        sig_block = SignatureBlock{
            algorithm:  SIG_TYPE_ED25519,
            key_index:  0xFF,  // Legacy single-key (OTA uses baked-in KI_public)
            sig_length: 64,
            reserved:   [0u8; 4],
            public_key: LEGACY_KI_PUBLIC,
            signature:  ota_signature,
            crc32:      0,  // Computed below
        }
        sig_block.crc32 = crc32_compute(serialize_sig_block_prefix(sig_block))

        // Write SignatureBlock to slot
        storage_write(slot, 80 + re_encrypted_payload.len + 16, serialize_sig_block(sig_block))

        // ── Step 8: Zeroize sensitive key material ──
        secure_zeroize(shared_secret)
        secure_zeroize(k_s)
        secure_zeroize(gcm_nonce)
        secure_zeroize(kd_storage)
        secure_zeroize(ko_private)
        secure_zeroize(plaintext)  // Plaintext firmware — must not persist in RAM

        // ── Step 9: Reload the now-standard image from slot ──
        // The slot now contains a normal SBOP image (KD_Storage encrypted).
        // Reload so Phase 5-6 operate on the re-encrypted image.
        image_data = storage_read_slot(slot)
        if image_data == null: return Err(ERR-STOR-READ-001)
        header = parse_header(image_data)
        if header == null: return Err(ERR-BOOT-PARSE-001)
        payload = image_data[header.header_size .. header.header_size + header.image_size]

        // FLAG_OTA_PENDING is now cleared; OTA verification already done.
        // Set flag to skip Phases 5-6 (signature + integrity already verified
        // on plaintext before KD_Storage re-encryption).
        ota_verified = true
        IWDG_REFRESH()
    else:
        // Non-OTA path: sig_block was parsed in Phase 4
        if sig_block == null: return Err(ERR-BOOT-PARSE-006)
        ota_verified = false
    // ── End of OTA Decrypt + Re-encrypt (Phase 4.5) ──

    // ── Phase 5: Verify Signature ──
    state = VERIFY_SIGNATURE
    IWDG_REFRESH()

    if !ota_verified:
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

        let sig_data = concat(image_data[0 .. header.header_size], payload)
        let sig_valid = ed25519_verify(sig_data, signature, public_key)
        if sig_valid != true: return Err(ERR-BOOT-CRYPTO-001)
        // FIH redundant verify (fault injection defense)
        let sig_valid_v2 = ed25519_verify(sig_data, signature, public_key)
        if sig_valid_v2 != true: return Err(ERR-BOOT-CRYPTO-001)
        IWDG_REFRESH()

    // ── Phase 6: Verify Integrity ──
    state = VERIFY_INTEGRITY

    if !ota_verified:
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
    else:
        // OTA path: integrity already verified in Phase 4.5 (GCM auth tag +
        // Ed25519 signature on plaintext). KD_Storage encrypted payload hash
        // does NOT match header.hash (which covers plaintext). Cross-check:
        // verify that KD_Storage decrypt + SHA-256 matches header.hash.
        let kd_storage = crypto_derive_key(KD, salt="", info="SBOP-STORAGE-v1", output_len=32)
        let storage_nonce = storage_read_iv(slot)
        let decrypted_check = aes_256_gcm_decrypt(
            key = kd_storage.unwrap(),
            nonce = storage_nonce,
            aad = header.firmware_version as [u8; 4] || (slot as u8),
            ciphertext = payload[0 .. payload.len - 16],
            tag = payload[payload.len - 16 .. payload.len]
        )
        if decrypted_check.is_err(): return Err(ERR-BOOT-CRYPTO-008)
        let computed_hash = crypto_compute_hash(decrypted_check.unwrap())
        if constant_time_compare(computed_hash, header.hash, 32) != true:
            return Err(ERR-BOOT-CRYPTO-002)
        secure_zeroize(kd_storage)
        secure_zeroize(decrypted_check)

    // IV reuse detection: check if this (KD, IV_dev) pair was used before.
    // Reads last_iv_hash from frequent journal (ring buffer, no sector erase).
    // If the truncated hash matches, the same keystream was used twice —
    // possible AES-CTR replay attack.
    let last_record = frequent_journal_read_last()
    if last_record.valid:
        let current_iv = storage_read_iv(slot)
        let current_iv_hash = sha256(current_iv || KD[0..16])
        if constant_time_compare(current_iv_hash[0..24], last_record.last_iv_hash, 24):
            return Err(ERR-BOOT-CRYPTO-007)  // IV reuse detected
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

    // ── Phase 9: Mark Slot Testing (test/confirm model) ──
    // New images enter TESTING state, not ACTIVE. The application must call
    // boot_set_confirmed() to promote to ACTIVE. If the device resets before
    // confirmation, the bootloader immediately reverts to the fallback slot —
    // no N-attempt retry, no boot_attempt_count accumulation.
    state = MARK_TESTING
    IWDG_REFRESH()
    let mark_ok = storage_set_slot_status(slot, TESTING)
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

    // Record this boot in the frequent journal (ring buffer, no sector erase).
    // prev_boot_phase=0x01 (reached EXECUTE), flags encode TESTING/FALLBACK state.
    // Also records IV hash for reuse detection and extends the measured boot PCR chain.
    // Critical journal is NOT written here — slot state unchanged.
    let boot_flags = 0x00
    if storage_get_slot_status(slot) == TESTING: boot_flags |= 0x01  // was_testing
    if storage_get_slot_status(slot) == FALLBACK: boot_flags |= 0x02 // was_fallback
    let iv_hash = sha256(current_iv || KD[0..16])

    // Measured boot: extend PCR chain for remote attestation.
    // PCR[n] = SHA-256(PCR[n-1] || image_hash || slot || boot_phase)
    // If no previous record exists (first boot), compute initial PCR.
    let last_record = frequent_journal_read_last()
    let prev_pcr = last_record.valid ? last_record.pcr_value : measured_boot_initial_pcr()
    let new_pcr = sha256(prev_pcr || header.hash || slot as u8 || 0x01)

    frequent_journal_append(0x01, boot_flags, iv_hash[0..24], new_pcr)

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

## 3. Application Confirm API

```
/// Called by the application after it has verified its own health.
/// Must be called before the next reset — typically early in application init,
/// after self-test, sensor check, and network connectivity are confirmed.
/// If not called before the next reset, the bootloader immediately reverts
/// to the fallback slot (test/confirm model — no N-attempt retry).
function boot_set_confirmed() -> Result<(), BootError>:
    // Only the active application image can confirm itself.
    // This is NOT a bootloader-phase function — it's called from Zone 2 (app).
    // Requires the application to have boot API access (SVC or direct call
    // before MPU lockdown).

    let active_slot = metadata_read_active_slot()
    let slot_status  = storage_get_slot_status(active_slot)

    if slot_status != TESTING:
        return Err(ERR-BOOT-STATE-006)  // Not in testing state — already confirmed?

    // Atomically transition TESTING → ACTIVE in critical journal.
    // This triggers a critical journal write (two-copy atomic, sector erase).
    // Infrequent operation — only once per successful OTA update.
    storage_set_slot_status(active_slot, ACTIVE)

    // Record confirmation in frequent journal (ring buffer append, no erase).
    // Uses current IV hash from the most recent boot record.
    let last_record = frequent_journal_read_last()
    let iv_hash = last_record.valid ? last_record.last_iv_hash : [0u8; 24]
    // Use current PCR (no new extend — confirmation doesn't change measurement)
    frequent_journal_append(0x02, 0x00, iv_hash, last_record.pcr_value)  // 0x02 = confirmed

    return OK
```

**Calling convention:** The application binary exports `boot_set_confirmed` as a weak symbol.
The bootloader provides a default implementation (SVC-based metadata write). The application
calls it once during early init, after health checks pass:

```c
// Application main.c — early init
int main(void) {
    hal_init();
    if (self_test_pass() && sensors_ok() && network_ready()) {
        boot_set_confirmed();  // Promote TESTING → ACTIVE
    }
    // ... normal application code ...
}
```

**Failure scenarios:**

| Scenario | Slot status before boot | Action |
|---|---|---|
| App confirms, then reboots normally | TESTING → boot_set_confirmed() → ACTIVE | Boot as ACTIVE |
| App crashes before confirming | TESTING (unconfirmed) | Immediate rollback to fallback |
| App confirms, new OTA arrives | ACTIVE → FALLBACK | Old confirmed image becomes fallback |
| Power loss before confirm | TESTING (unconfirmed) | Immediate rollback — no retry |
| App confirms twice | ACTIVE (already confirmed) | Returns ERR-BOOT-STATE-006 |

---

## 4. Fail-Safe Handler

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
    
    // Enter Serial Recovery Protocol (SRP) event loop.
    // SRP handles PING, AUTH_CHALLENGE, AUTH_RESPONSE, QUERY_STATUS,
    // UPLOAD_FIRMWARE, ERASE_SLOT, RESET_DEVICE, GET_ERROR_LOG.
    // IWDG is still active — must refresh to prevent 800 ms reset.
    // SRP frames are processed on USART1 (PA9/PA10).
    // See Serial_Recovery_Protocol.md for full protocol specification.
    loop:
        IWDG_REFRESH()
        
        // Send FAILSAFE beacon every 500 ms (rate-limited inside srp_beacon_send)
        srp_beacon_send(error.code, boot_slot, fallback_slot)
        
        // Process incoming SRP frames with 100 ms timeout
        if srp_frame_available(timeout_ms = 100):
            IWDG_REFRESH()
            srp_process_frame()
        
        // If recovery upload completes and device is reset via SRP RESET_DEVICE,
        // this loop never returns to the next iteration.
        
        platform_idle()  // WFI — USART1 RX interrupt wakes
```

---

## 5. SRP Recovery Boot

The Serial Recovery Protocol (see `Serial_Recovery_Protocol.md`) runs within the FAILSAFE handler. It provides authenticated firmware upload, slot management, and device control over USART1. Key integration points:

```
// SRP event processing (called from FAILSAFE loop):
function srp_process_frame():
    let frame = srp_frame_read()
    if frame.crc_invalid:
        return  // Discard silently — CRC-16 mismatch
    if frame.sequence != expected_seq:
        return  // Out-of-order — discard

    expected_seq = (expected_seq + 1) & 0xFF

    match frame.opcode:
        case PING:
            srp_send_response(PING, frame.payload)  // Echo

        case GET_VERSION:
            srp_send_response(GET_VERSION, build_version_payload())

        case AUTH_CHALLENGE:
            if srp_brute_force_locked_out():
                srp_send_error(ERR-SRP-AUTH-003)
                return
            let challenge = crypto_random_bytes(32)
            srp_auth_session.challenge = challenge
            srp_send_response(AUTH_CHALLENGE, challenge || device_uid)

        case AUTH_RESPONSE:
            let hmac_host = frame.payload[0..32]
            let access_level = frame.payload[33]
            let expected = hmac_sha256(KD_Debug,
                           srp_auth_session.challenge || 0x02)
            if constant_time_compare(hmac_host, expected, 32):
                srp_auth_session.authenticated = true
                srp_auth_session.access_level = access_level
                srp_auth_session.expires_at = rtc_get_seconds() + 300
                srp_brute_force_reset()
                srp_send_response(AUTH_RESPONSE, [access_level, 300])
            else:
                srp_brute_force_record_failure()
                srp_send_error(ERR-ID-AUTH-001)

        case QUERY_STATUS if auth_check():
            srp_send_response(QUERY_STATUS, build_status_payload())

        case UPLOAD_FIRMWARE if auth_check():
            srp_handle_upload(frame)

        case ERASE_SLOT if auth_check():
            srp_handle_erase(frame)

        case RESET_DEVICE if auth_check():
            srp_handle_reset(frame)

        case GET_ERROR_LOG if auth_check():
            srp_handle_get_log(frame)

        case _:
            srp_send_error(ERR-SRP-CMD-001)

function auth_check() -> bool:
    if not srp_auth_session.authenticated:
        srp_send_error(ERR-SRP-AUTH-001)
        return false
    if rtc_get_seconds() > srp_auth_session.expires_at:
        srp_auth_session.authenticated = false
        srp_send_error(ERR-SRP-AUTH-002)
        return false
    return true
```

**SRP boot flow integration:**

```
Boot fails (both slots dead)
  └─→ boot_enter_failsafe(ERR-BOOT-STATE-004)
        └─→ handle_boot_error()
              └─→ FAILSAFE loop with SRP
                    ├─→ Host connects, authenticates
                    ├─→ Host uploads firmware via UPLOAD_FIRMWARE
                    ├─→ Host sends RESET_DEVICE
                    └─→ Device resets, boots new firmware
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
| PARSE_HEADER | FLAG_OTA_PENDING set | OTA_DECRYPT | OTA image detected |
| PARSE_HEADER | all fields valid, not OTA | VERIFY_SIGNATURE | Section 2 rules R-IMG-001..006 |
| PARSE_HEADER | any field invalid | FAILSAFE | ERR-BOOT-PARSE-001..006 |
| OTA_DECRYPT | ECDH + decrypt + re-encrypt OK | VERIFY_SIGNATURE | OTA verified = true |
| OTA_DECRYPT | GCM auth failure | FAILSAFE | ERR-BOOT-CRYPTO-008 |
| OTA_DECRYPT | Signature invalid on plaintext | FAILSAFE | ERR-BOOT-CRYPTO-001 |
| VERIFY_SIGNATURE | signature valid (or OTA skip) | VERIFY_INTEGRITY | crypto_verify_signature == true |
| VERIFY_SIGNATURE | signature invalid | FAILSAFE | ERR-BOOT-CRYPTO-001 |
| VERIFY_INTEGRITY | hash match | CHECK_VERSION | constant_time_compare == true |
| VERIFY_INTEGRITY | hash mismatch | FAILSAFE | ERR-BOOT-CRYPTO-002 |
| CHECK_VERSION | version >= stored | COMMIT_VERSION | firmware_version >= stored_version |
| CHECK_VERSION | rollback detected | FAILSAFE | ERR-BOOT-VERSION-001 |
| COMMIT_VERSION | write OK | MARK_TESTING | storage_write returns OK |
| COMMIT_VERSION | write error | FAILSAFE | ERR-STOR-WRITE-001 |
| MARK_TESTING | mark OK | LOCK_BOOT | slot status set to TESTING |
| MARK_TESTING | mark error | FAILSAFE | ERR-STOR-WRITE-001 |
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
| I7 | secure_zeroize(K_s, SharedSecret, plaintext) in Phase 4.5 | OTA session keys zeroized immediately after re-encrypt |
| I8 | FLAG_OTA_PENDING cleared after successful re-encrypt | One-time OTA decrypt — subsequent boots use KD_Storage fast path |

---

## 10. Timing Budgets

**Pass 1 (IWDG-protected, single slot verification):**

| Phase | Maximum Allowed Time | Notes |
| --- | --- | --- |
| INIT (incl. IWDG start + NOR detect) | ~6 ms | Platform init + JEDEC ID read |
| SELECT_SLOT | < 1 ms | Critical journal read (128 B) + frequent journal read (32 B) |
| LOAD_IMAGE | < 100 ms | Depends on image size and flash speed |
| PARSE_HEADER | < 1 ms | Fixed-size parse |
| OTA_DECRYPT (X25519 + GCM + re-encrypt) | < 200 ms | ECDH + HKDF + AES-256-GCM decrypt + KD_Storage re-encrypt + flash write |
| VERIFY_SIGNATURE | < 50 ms | Ed25519 verify (skipped if OTA already verified) |
| VERIFY_INTEGRITY | < 100 ms | SHA-256 over payload + IV reuse check |
| CHECK_VERSION | < 1 ms | Simple integer comparison |
| COMMIT_VERSION | < 10 ms | SR1 OTP write (new version only) |
| MARK_TESTING | < 45 ms | Critical journal write (OTA events only) — sector erase + 128 B page program |
| LOCK_BOOT | < 1 ms | Memory protection enable + BP bit set |
| EXECUTE (incl. frequent journal append) | < 2 ms | Frequent journal 32 B page program (~0.7 ms) + RTC magic + NVIC_SystemReset |

**Pass 2 (IWDG stopped, fast path to application):**

| Phase | Maximum Allowed Time | Notes |
| --- | --- | --- |
| INIT (detect RTC magic) | ~2 ms | Platform init (minimal) |
| Jump to prepared application | ~3 ms | VTOR set, workspace clear, MSP+PC load |

**Total boot budget target (non-OTA):** < 200 ms (Pass 1) + ~5 ms (Pass 2) ≈ < 205 ms from cold reset to firmware execution.
**Total boot budget target (OTA first boot):** < 400 ms (Pass 1 with Phase 4.5 ECDH + GCM decrypt + KD_Storage re-encrypt) + ~5 ms (Pass 2) ≈ < 405 ms.

The OTA first boot exceeds the 300 ms SBOP general target; this is acceptable because OTA first boot is a one-time event per update. Subsequent boots of the same image use the KD_Storage-encrypted fast path (~205 ms).

---

## 11. Metadata Journal API

The metadata journal is split into two tiers with different update frequencies and write protocols. The critical journal (slot state, version ceiling) uses two-copy atomic writes and is updated only on OTA/slot transitions. The frequent journal (per-boot counters, IV hash) uses a simple ring buffer append with no sector erase until the ring wraps.

### 11.1 Critical Journal API

```
/// Read the authoritative copy of the critical journal.
/// Determines active copy by comparing sequence numbers and CRC-32.
/// Returns the critical metadata: active_slot, boot_target, re_encrypt_state,
/// slot_a_info, slot_b_info, max_allowed_version.
function critical_journal_read() -> CriticalMetadata:
    let copy_a = nor_spi_read(0x20_0000, 128)
    let copy_b = nor_spi_read(0x20_1000, 128)

    let seq_a = copy_a[0..4] as u32
    let seq_b = copy_b[0..4] as u32
    let crc_a_ok = (crc32(copy_a[0..124]) == copy_a[124..128] as u32)
    let crc_b_ok = (crc32(copy_b[0..124]) == copy_b[124..128] as u32)

    if crc_a_ok and crc_b_ok:
        if seq_a >= seq_b:
            let was_committed = (seq_a & 1) == 0  // even = committed
        else:
            let was_committed = (seq_b & 1) == 0
        if not was_committed:
            // Odd sequence = power loss during write.
            // Discard partial; use the other (previously committed) copy.
            return (seq_a > seq_b) ? parse_metadata(copy_b) : parse_metadata(copy_a)
        return (seq_a >= seq_b) ? parse_metadata(copy_a) : parse_metadata(copy_b)

    if crc_a_ok: return parse_metadata(copy_a)
    if crc_b_ok: return parse_metadata(copy_b)
    // Neither valid — metadata destroyed
    boot_enter_failsafe(ERR-STOR-READ-001)


/// Atomically write the critical journal using the two-copy protocol.
/// Only called on OTA/slot events — NOT every boot.
function critical_journal_write(new_data: &CriticalMetadata):
    let (active_offset, inactive_offset) = critical_journal_find_active()
    let current_seq = nor_spi_read_u32(active_offset)

    // Write to INACTIVE copy (separate 4 KB sector — safe to erase)
    let new_seq = current_seq + 1  // odd = in-progress
    new_data[0..4] = new_seq
    new_data[124..128] = crc32(new_data[0..124])

    nor_spi_erase_sector(inactive_offset)
    nor_spi_page_program(inactive_offset, new_data, 128)

    // Verify write integrity before commit
    let verify = nor_spi_read(inactive_offset, 128)
    assert(crc32(verify[0..124]) == verify[124..128] as u32, ERR-STOR-WRITE-001)

    // Atomic commit: increment sequence to even
    nor_spi_write_u32(inactive_offset, new_seq + 1)  // even = committed


function critical_journal_find_active() -> (u32, u32):
    let seq_a = nor_spi_read_u32(0x20_0000)
    let seq_b = nor_spi_read_u32(0x20_1000)
    if seq_a >= seq_b:
        return (0x20_0000, 0x20_1000)  // (active, inactive)
    else:
        return (0x20_1000, 0x20_0000)
```

### 11.2 Frequent Journal API

```
/// Record size: 64 bytes. 64 records per 4 KB sector.
/// Sector at 0x20_2000.
const FREQUENT_JOURNAL_BASE: u32 = 0x20_2000
const FREQUENT_RECORD_SIZE: u32 = 64
const FREQUENT_SECTOR_SIZE: u32 = 4096
const FREQUENT_MAX_RECORDS: u32 = 64

struct FrequentRecord:
    boot_seq:        u32    // [0]  Monotonic boot counter
    prev_boot_phase: u8     // [4]  0x00=not reached EXECUTE, 0x01=EXECUTE, 0x02=confirmed
    flags:           u8     // [5]  bit0=was_testing, bit1=was_fallback
    reserved:        u8[2]  // [6]  Must be zero
    last_iv_hash:    u8[24] // [8]  Truncated SHA-256(IV_dev || KD[0..16])
    pcr_value:       u8[32] // [32] SHA-256 chained measurement (measured boot, see §11.3)
    crc16:           u16    // [62] CRC-16-CCITT over bytes 0..61


/// Append a new record to the frequent journal ring buffer.
/// No sector erase unless the ring wraps (once per ~64 boots).
/// Called once per successful boot in Phase 11 (EXECUTE).
/// The PCR value is the chained measurement for remote attestation.
function frequent_journal_append(phase: u8, flags: u8, iv_hash: &[u8; 24],
                                 pcr_value: &[u8; 32]):
    let last = frequent_journal_read_last()
    let new_seq = (last.valid) ? last.boot_seq + 1 : 1

    let write_off = frequent_journal_find_next_free()
    if write_off >= FREQUENT_SECTOR_SIZE:
        // Ring full — erase and restart from beginning
        nor_spi_write_enable()
        nor_spi_erase_sector(FREQUENT_JOURNAL_BASE)
        nor_spi_wait_busy()
        write_off = 0

    // Build record (CRC-16 over bytes 0..61)
    let mut record = [0u8; 64]
    record[0..4]   = new_seq as u32
    record[4]      = phase
    record[5]      = flags
    record[6..8]   = [0u8, 0u8]
    record[8..32]  = iv_hash[0..24]
    record[32..64] = pcr_value[0..32]
    let crc = crc16_ccitt(record[0..62])
    record[62..64] = crc as u16

    nor_spi_page_program(FREQUENT_JOURNAL_BASE + write_off, record, 64)

    // Verify
    let verify = nor_spi_read(FREQUENT_JOURNAL_BASE + write_off, 64)
    assert(memcmp(record, verify, 64) == 0, ERR-STOR-WRITE-001)


/// Read the most recent valid record from the frequent journal.
/// Scans backward from end of sector for first non-erased record with valid CRC-16.
function frequent_journal_read_last() -> Option<FrequentRecord>:
    for off in (FREQUENT_SECTOR_SIZE - FREQUENT_RECORD_SIZE) down to 0
             step FREQUENT_RECORD_SIZE:
        let data = nor_spi_read(FREQUENT_JOURNAL_BASE + off, 64)
        if data[0..4] != 0xFFFFFFFF:  // Not erased
            if crc16_ccitt(data[0..62]) == data[62..64] as u16:
                return Some(FrequentRecord{
                    boot_seq:        data[0..4] as u32,
                    prev_boot_phase: data[4],
                    flags:           data[5],
                    last_iv_hash:    data[8..32],
                    pcr_value:       data[32..64],
                })
    return None  // Sector empty or all corrupt


/// Find the next free record slot (first erased 64-byte aligned offset).
function frequent_journal_find_next_free() -> u32:
    for off in 0..FREQUENT_SECTOR_SIZE step FREQUENT_RECORD_SIZE:
        let header = nor_spi_read(FREQUENT_JOURNAL_BASE + off, 4)
        if header == 0xFFFFFFFF:
            return off
    return FREQUENT_SECTOR_SIZE  // Sector full


/// Erase the frequent journal sector (called on slot fallback to start fresh).
function frequent_journal_erase():
    nor_spi_write_enable()
    nor_spi_erase_sector(FREQUENT_JOURNAL_BASE)
    nor_spi_wait_busy()
```

### 11.3 Measured Boot PCR Chain

```
/// Compute the initial PCR value for the measured boot chain.
/// PCR[0] = SHA-256("SBOP_MEASURED_BOOT_V1" || bootloader_version || device_uid)
/// This is deterministic — given the same bootloader version and device UID,
/// the initial PCR is always the same. It can be recomputed by the verifier.
function measured_boot_initial_pcr() -> [u8; 32]:
    let device_id = identity_get_device_id()
    let input = "SBOP_MEASURED_BOOT_V1" || BOOTLOADER_VERSION as u32 || device_id.uid
    return sha256(input)


/// Extend the PCR chain with a new boot event.
/// PCR[n] = SHA-256(PCR[n-1] || image_hash || slot || boot_phase)
/// Called in Phase 11 before frequent_journal_append().
function measured_boot_extend(prev_pcr: &[u8; 32], image_hash: &[u8; 32],
                              slot: SlotID, boot_phase: u8) -> [u8; 32]:
    let mut input = [0u8; 32 + 32 + 1 + 1]  // 66 bytes
    input[0..32]   = prev_pcr
    input[32..64]  = image_hash
    input[64]      = slot as u8
    input[65]      = boot_phase
    return sha256(input)


/// Attestation verification (performed by remote verifier):
///
///   let pcr = SHA-256("SBOP_MEASURED_BOOT_V1" || bl_version || device_uid)
///   for each record in measurement_log:
///       pcr = SHA-256(pcr || record.image_hash || record.slot || record.boot_phase)
///   assert(pcr == device.current_pcr_value)
///
/// Any modification to a measurement record changes its pcr_value, which
/// breaks the chain for all subsequent records. The verifier detects this
/// by recomputing the chain from the initial PCR.
```

### 11.3 Journal Write Triggers

| Event | Critical Journal | Frequent Journal |
|-------|-----------------|-----------------|
| Normal boot (Phase 11 EXECUTE) | **Not written** | `append(0x01, flags, iv_hash, pcr)` |
| OTA writes new image to slot | Write (slot status → PENDING) | Not written |
| OTA decrypt + re-encrypt (Phase 4.5) | Write (re_encrypt_state → COMMITTED, slot data) | Not written |
| First boot after OTA (Phase 9) | Write (slot status → TESTING) | Not written |
| `boot_set_confirmed()` | Write (slot status → ACTIVE) | `append(0x02, 0x00, iv_hash)` |
| Slot verification fails | Write (slot status → INVALID) | Not written |
| Slot fallback (SELECT_SLOT) | Write (boot_target, active_slot) | `erase()` (fresh start for fallback slot) |
| OTA updates version ceiling | Write (max_allowed_version) | Not written |

**Write amplification comparison:**

| Journal | Erases per year (daily boot, weekly OTA) | Erases per year (hourly boot, daily OTA) |
|---------|------------------------------------------|----------------------------------------|
| Single journal (old) | 365 | 8,760 |
| Critical journal (new) | ~52 (OTA events only) | ~365 (OTA events only) |
| Frequent journal (new) | ~3 (128 boots/erase) | ~69 (128 boots/erase) |
| **Total (new)** | **~55** | **~434** |
| **Reduction vs old** | **85%** | **95%** |
