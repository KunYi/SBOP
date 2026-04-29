# Boot State Machine - Detailed Specification

**Document ID:** SUB-BOOT-ST-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines detailed behavior of each boot state, aligned with the 12-phase boot flow defined in `Boot_Flow_Pseudocode.md` and `State_Machine.md`.

---

## 2. State Definitions

### RESET

- Entry point after power-on or reset
- Hardware vectors to bootloader entry
- No trust assumed; registers in reset state

---

### INIT

- Initialize MPU/MMU, system clocks, and crypto engine
- TRNG health check (repetition count, adaptive proportion)
- Configure watchdog for tight boot timeout (1-5s)
- Load KI public key from Zone 1 read-only region

Failure: → FAILSAFE

---

### SELECT_SLOT

- Read slot A and slot B metadata
- Determine validity: signature OK, integrity OK, version >= OTP counter
- Select highest-version valid slot
- If both valid and versions equal: use currently marked ACTIVE
- If neither valid: → FAILSAFE

Failure: → FAILSAFE

---

### LOAD_IMAGE

- Load ImageHeader from selected slot into RAM
- Validate magic number (0x53424F50)
- Validate header version, header_size, image_size bounds

Failure: → FAILSAFE (or try other slot if not yet attempted)

---

### PARSE_HEADER

- Parse ImageHeader struct fields: firmware_version, flags, hash, reserved
- Validate flags field (no unsupported bits set)
- Validate reserved field is zero
- Extract signature algorithm from header flags

If FLAG_OTA_PENDING: → OTA_DECRYPT
Otherwise: → VERIFY_SIGNATURE
Failure: → FAILSAFE

---

### OTA_DECRYPT

- Parse OTAImagePackage header (OTAHeader 44 B)
- X25519 ECDH key agreement: KO_private (secure element) × ephemeral public key (from OTA header)
- HKDF derive K_s ("SBOP-OTA-v1" || FirmwareVersion) and GCM nonce ("SBOP-OTA-NONCE-v1")
- AES-256-GCM decrypt payload with K_s, nonce, AAD
- Verify Ed25519 signature on decrypted plaintext (FIH redundant verify)
- Re-encrypt plaintext with KD_Storage (AES-256-GCM, device-unique)
- Write re-encrypted image to slot, clear FLAG_OTA_PENDING
- Zeroize K_s, SharedSecret, plaintext, KO_private

Success: → VERIFY_SIGNATURE (ota_verified = true)
Failure: → FAILSAFE

---

### VERIFY_SIGNATURE

- Verify Ed25519 signature against KI public key
- Signature covers ImageHeader + ImageBody
- Constant-time verification (no early exit)

Success: → VERIFY_INTEGRITY
Failure: → FAILSAFE

---

### VERIFY_INTEGRITY

- Compute SHA-256 over ImageBody
- Compare with hash stored in ImageHeader
- Use `constant_time_compare()` — must be timing-invariant

Success: → CHECK_VERSION
Failure: → FAILSAFE

---

### CHECK_VERSION

- Compare `image_header.firmware_version` against OTP monotonic counter
- Version must be >= OTP counter value
- Redundant comparison (two independent reads of OTP) for glitch resistance

Success: → COMMIT_VERSION (if version > stored) or → MARK_TESTING (if version == stored)
Failure: → FAILSAFE

---

### COMMIT_VERSION

- Write stored version counter to new version (only if `image_version > stored_version`)
- Write must complete atomically (critical journal two-copy protocol)
- Verify write succeeded by reading back
- Retry once on failure; then → FAILSAFE

Success: → MARK_TESTING
Failure: → FAILSAFE

---

### MARK_TESTING

- Set selected slot state to TESTING (test/confirm model)
- Set other slot state to FALLBACK (if it was ACTIVE)
- Write slot metadata atomically (critical journal)
- Application must call boot_set_confirmed() to promote to ACTIVE
- If device resets before confirmation → immediate rollback

Success: → LOCK_BOOT
Failure: → FAILSAFE

---

### LOCK_BOOT

- Configure MPU/MMU: Zone 1 read-only, Zone 2 no access to Zone 1 regions
- Verify MPU configuration is active
- Disable debug port (unless authenticated debug session)
- Lock security fuses

Success: → EXECUTE
Failure: → FAILSAFE

---

### EXECUTE

- Set program counter to Zone 2 entry point
- Jump to application firmware
- Boot responsibility ends — watchdog handed off to application

No transitions from EXECUTE (boot is complete).

---

### FAILSAFE

- Terminal state — no firmware execution
- All outputs set to safe state
- Watchdog holds device in reset or safe loop
- Recovery: requires valid firmware update via minimal bootloader or service tool

---

## 3. State Invariants

| # | Invariant |
|---|-----------|
| I1 | No state may skip any verification step |
| I2 | No direct transition to EXECUTE without passing all 12 phases |
| I3 | All failures converge to FAILSAFE |
| I4 | MPU/MMU must be configured before EXECUTE |
| I5 | Zone 1 memory locked read-only before EXECUTE |

---

## 4. Timing Constraints

| Phase | Maximum Time | Notes |
|-------|-------------|-------|
| RESET → INIT | Hardware-dependent | Includes power ramp, clock stabilization |
| INIT → SELECT_SLOT | 6 ms | Crypto engine init, TRNG health test |
| SELECT_SLOT | < 1 ms | Critical + frequent journal reads |
| LOAD_IMAGE | 100 ms | Flash read, bounded by image size |
| PARSE_HEADER | < 1 ms | Fixed-size struct parsing |
| OTA_DECRYPT | < 200 ms | X25519 ECDH + AES-256-GCM + KD_Storage re-encrypt (OTA first boot only) |
| VERIFY_SIGNATURE | 50 ms | Ed25519 (skipped if OTA already verified) |
| VERIFY_INTEGRITY | 100 ms | SHA-256 over full image (size-dependent) |
| CHECK_VERSION | < 1 ms | Version counter read |
| COMMIT_VERSION | < 10 ms | Version counter write (if needed) |
| MARK_TESTING | < 45 ms | Critical journal write (OTA events only) |
| LOCK_BOOT | < 1 ms | MPU register writes |
| EXECUTE | < 2 ms | Frequent journal append + RTC magic + system reset |

---

## 5. References

| Document | Reference |
|----------|-----------|
| Boot Flow Pseudocode | `Boot_Flow_Pseudocode.md` |
| Boot Overview | `Boot_Overview.md` |
| State Machine | `../../02_System_Design/State_Machine.md` |
| Trust Model | `../../00_Architecture/Trust_Model.md` |
