# SBOP Data Structure Definitions

**Document ID:** SYS-DS-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

This document defines the logical data structures exchanged between SBOP components. All structures are defined at the specification level — concrete C/Rust/Zig types are implementation decisions derived from these definitions.

---

## 2. Conventions

- All multi-byte fields are **little-endian** unless specified otherwise
- All structures are **4-byte aligned**
- Reserved fields must be **zero-filled**
- Enum discriminants are `u8` unless a larger range is needed
- Length fields are `u32` unless constrained to a known maximum
- Strings are **UTF-8, null-terminated** unless noted

---

## 3. ImageHeader

```rust
/// Firmware image header. Fixed-size, parsed at boot.
/// Total size: 80 bytes (header_size field = 80)
struct ImageHeader {
    magic:            u32,      // 0x53424F50 ("SBOP")
    header_version:   u16,      // Header format version (2)
    header_size:      u16,      // Total header size in bytes (= 80)
    image_size:       u32,      // Payload size in bytes
    firmware_version: u32,      // Monotonic version number
    flags:            u32,      // Feature flags (see Section 3.1)
    hash:             [u8; 32], // SHA-256 of firmware payload
    enc_key_index:    u8,       // KO key pair index for OTA ECDH (0..3, 0xFF = unused)
    reserved:         [u8; 3],  // Must be zero
    header_hmac:      [u8; 16], // HMAC-SHA-256-128(KD, header[0..56]) — SPI bus integrity
    reserved2:        [u8; 8],  // Must be zero (future use)
}
```

### 3.1 Flags Field

| Bit | Name | Description |
| --- | --- | --- |
| 0 | FLAG_ENCRYPTED | Payload is encrypted |
| 1 | FLAG_COMPRESSED | Payload is compressed |
| 2 | FLAG_DELTA | Image is a delta/patch update |
| 3-7 | Reserved | Must be zero |
| 8 | FLAG_PQC_SIGNATURE | Signature uses post-quantum algorithm |
| 9 | FLAG_OTA_PENDING | First boot after OTA download. Payload is AES-256-GCM encrypted with K_s (from X25519 ECDH). Bootloader must: (1) X25519 ECDH with ephemeral pubkey from OTAImagePackage + KO_public → SharedSecret, (2) HKDF → K_s, (3) AES-256-GCM decrypt → plaintext SBOP image, (4) verify Ed25519 signature on plaintext, (5) re-encrypt with KD_Storage for at-rest protection. |
| 10-11 | Reserved | Must be zero |
| 12-15 | enc_key_index (in-band) | OTA encryption key pair index. Redundant with header.enc_key_index field — cross-checked at boot. |
| 16-31 | Reserved | Must be zero |

### 3.2 Validation Rules

| Rule | Description |
| --- | --- |
| R-IMG-001 | `magic` must == 0x53424F50 |
| R-IMG-002 | `header_version` must be ≤ MAX_SUPPORTED_HEADER_VERSION |
| R-IMG-003 | `header_size` must be ≥ 80 |
| R-IMG-004 | `image_size` must be > 0 and ≤ MAX_IMAGE_SIZE |
| R-IMG-005 | `firmware_version` must be > 0 (version 0 is invalid) |
| R-IMG-006 | Reserved bytes must be zero |
| R-IMG-007 | SHA-256 of payload must match `hash` |
| R-IMG-008 | `header_hmac` must == HMAC-SHA-256(KD, header[0..56]) truncated to 128 bits — protects against SPI bus tampering of plaintext header fields |
| R-IMG-009 | If `enc_key_index` in flags bits 12-15 differs from `header.enc_key_index`, reject image (redundant check) |

---

## 4. OTAImagePackage

The OTA Image Package is the transport format for firmware updates. It wraps the SBOP image (ImageHeader + Firmware + SignatureBlock) with X25519 ECDH + AES-256-GCM encryption for confidentiality during transmission. This is the format produced by the Development Team packaging process and downloaded by devices.

```rust
/// OTA Image Package — transport format for encrypted firmware updates.
/// Produced by: Development Team packaging process (sign-then-encrypt).
/// Consumed by: Device bootloader Phase 5b (decrypt + re-encrypt).
struct OTAImagePackage {
    // ── OTA Header (44 bytes fixed) ──
    magic:            u32,       // 0x534F5441 ("SOTA")
    version:          u32,       // OTA package format version (1)
    firmware_version: u32,       // Monotonic firmware version number
    timestamp:        u64,       // Unix timestamp of packaging
    ephemeral_pubkey: [u8; 32],  // X25519 ephemeral public key (per-release)
    payload_length:   u32,       // Length of Encrypted Payload in bytes
    aad_length:       u16,       // Length of AAD field in bytes (0..512)
    reserved:         [u8; 6],   // Must be zero

    // ── AAD (variable, aad_length bytes) ──
    aad:              [u8; aad_length],  // Additional Authenticated Data (GCM)

    // ── Encrypted Payload (variable, payload_length bytes) ──
    encrypted_payload: [u8; payload_length],
    // After AES-256-GCM decryption, contains:
    //   SBOP Image = ImageHeader (80 B) || Firmware Binary || SignatureBlock (116 B)

    // ── GCM Authentication Tag (16 bytes) ──
    gcm_tag:          [u8; 16],   // AES-256-GCM authentication tag

    // ── Ed25519 Signature (64 bytes) ──
    signature:        [u8; 64],   // Ed25519 signature over plaintext firmware
}
```

### 4.1 OTA Header Fields

| Field | Size | Description |
| --- | --- | --- |
| magic | 4 B | `0x534F5441` ("SOTA") — identifies this as an SBOP OTA image package |
| version | 4 B | OTA package format version (current = 1) |
| firmware_version | 4 B | Monotonic firmware version — must be > device's current version |
| timestamp | 8 B | Unix timestamp of packaging — informational, not security-critical |
| ephemeral_pubkey | 32 B | X25519 ephemeral public key generated per release |
| payload_length | 4 B | Length of encrypted_payload in bytes |
| aad_length | 2 B | Length of AAD field in bytes (0..512, 0 = no AAD) |
| reserved | 6 B | Must be zero |

### 4.2 Encryption Parameters

```
Ephemeral_Keypair = X25519_KeyGen()          // Fresh per OTA release
SharedSecret      = X25519(Ephemeral_Private, KO_public)
K_s               = HKDF-Expand(SharedSecret, "SBOP-OTA-v1" || FirmwareVersion, 32)
GCM_Nonce         = HKDF-Expand(SharedSecret, "SBOP-OTA-NONCE-v1", 12)

AAD = firmware_version as u32 || device_class || timestamp as u64

EncryptedPayload = AES-256-GCM-Seal(
    key   = K_s,
    nonce = GCM_Nonce,
    aad   = AAD,
    data  = SBOP_Image  // ImageHeader || Firmware || SignatureBlock
)
```

### 4.3 OTA Package Validation Rules

| Rule | Description |
| --- | --- |
| R-OTA-001 | `magic` must == 0x534F5441 |
| R-OTA-002 | `version` must be ≤ MAX_SUPPORTED_OTA_VERSION |
| R-OTA-003 | `payload_length` must be > 0 and ≤ MAX_IMAGE_SIZE |
| R-OTA-004 | `aad_length` must be ≤ 512 |
| R-OTA-005 | `reserved` bytes must be zero |
| R-OTA-006 | GCM tag must authenticate (ciphertext + AAD integrity) |
| R-OTA-007 | Ed25519 `signature` must verify against decrypted plaintext firmware |
| R-OTA-008 | `firmware_version` > device's current active version (anti-rollback at OTA layer) |

### 4.4 Relationship to SBOP ImageHeader

The OTA Image Package is the **transport envelope**. After GCM decrypt, the plaintext is a standard SBOP Image:

```
OTA Image Package                     SBOP Image (after decrypt)
┌──────────────────────┐             ┌──────────────────────┐
│ OTAHeader            │             │ ImageHeader (80 B)   │
│ AAD                  │             │ Firmware Binary      │
│ EncryptedPayload ────┼─ decrypt ─→ │ SignatureBlock       │
│ GCM Tag              │             └──────────────────────┘
│ Ed25519 Signature ───┼─ verify ──→ (over plaintext above)
└──────────────────────┘
```

The SBOP ImageHeader.`FLAG_OTA_PENDING` (bit 9) is set when the image was installed via OTA. The bootloader detects this flag in Phase 5 and performs the ECDH decrypt → KD_Storage re-encrypt sequence before proceeding to signature verification on the recovered plaintext.

---

## 5. SignatureBlock

```rust
/// Signature block appended after firmware payload.
/// Carries the public key in-band so bootloader doesn't need pre-loaded keys.
/// Total size: 116 bytes
struct SignatureBlock {
    algorithm:    u8,          // Algorithm identifier (see §4.1)
    key_index:    u8,          // Key slot index (0-3) used to sign. 0xFF = single-key
    sig_length:   u16,         // Actual signature length in bytes
    reserved:     [u8; 4],     // Must be zero
    public_key:   [u8; 32],    // Ed25519 public key (not secret, embedded in image)
    signature:    [u8; 72],    // Signature bytes (right-padded with zeros)
    crc32:        u32,         // CRC-32 over all preceding fields
}
```

### 5.1 Algorithm Values

| Value | Algorithm | Signature Size |
| --- | --- | --- |
| 0x02 | Ed25519 | 64 bytes |

### 5.2 Key Index Values

| Value | Meaning |
| --- | --- |
| 0x00 | Key slot 0 (primary signing key) |
| 0x01 | Key slot 1 (secondary / rotation) |
| 0x02 | Key slot 2 (backup) |
| 0x03 | Key slot 3 (backup) |
| 0xFF | Single-key legacy mode (public key baked into bootloader .rodata) |

### 5.3 Validation Rules

| Rule | Description |
| --- | --- |
| R-SIG-001 | `algorithm` must be a recognized value (0x02) |
| R-SIG-002 | `key_index` must be 0-3 or 0xFF |
| R-SIG-003 | `sig_length` must match expected size for algorithm (64 for Ed25519) |
| R-SIG-004 | `reserved` fields must be zero |
| R-SIG-005 | `crc32` must match computed CRC-32 of preceding fields |
| R-SIG-006 | SHA-256(`public_key`) must match OTP fingerprint for `key_index` |
| R-SIG-007 | `key_index` slot must not be revoked (see key revocation bitmap) |
| R-SIG-008 | Signature must be cryptographically valid for `public_key` |

---

## 6. ImageInfo

```rust
/// Metadata about a firmware image in a slot.
/// Stored in the critical metadata journal (updated on slot transitions only).
/// boot_count is a cumulative count — it reflects the total boots at the time
/// of the last slot state transition, not the current per-boot count.
/// Per-boot counters are maintained in the frequent journal (FrequentRecord).
struct ImageInfo {
    slot:             SlotID,    // Which slot this image is in
    status:           SlotStatus,// Current state of the slot
    firmware_version: u32,       // Version of firmware in this slot
    image_hash:       [u8; 32],  // SHA-256 of payload
    flags:            u32,       // Feature flags from header
    build_timestamp:  u64,       // Unix timestamp of build (optional)
    boot_count:       u32,       // Cumulative boot count at last slot transition
}
```

---

## 7. SlotID and SlotStatus

```rust
enum SlotID: u8 {
    SLOT_A = 0x00,   // Primary slot
    SLOT_B = 0x01,   // Secondary slot
    SLOT_INVALID = 0xFF,
}

enum SlotStatus: u8 {
    EMPTY       = 0x00,  // Slot contains no image
    PENDING     = 0x01,  // Image ready for boot verification
    VERIFYING   = 0x02,  // Boot verification in progress
    TESTING     = 0x03,  // Image is active but unconfirmed (first boot after update)
    ACTIVE      = 0x04,  // Image is the active confirmed firmware
    FALLBACK    = 0x05,  // Image is active but previous version (after rollback)
    INVALID     = 0x06,  // Image failed verification
    CORRUPT     = 0x07,  // Storage corruption detected in this slot
}
```

### 7.1 Slot State Transitions

```
EMPTY ──(write)──> PENDING
PENDING ──(verify OK, first boot)──> TESTING
PENDING ──(verify fail)──> INVALID
TESTING ──(app confirms health)──> ACTIVE
TESTING ──(reset without confirm)──> INVALID  // Immediate rollback — no retry count
ACTIVE ──(new update to other slot)──> FALLBACK
ACTIVE ──(corruption detected)──> CORRUPT
FALLBACK ──(replaced by new update)──> Pending
INVALID ──(new write)──> PENDING
CORRUPT ──(reformat)──> EMPTY
```

**Key difference from N-attempt policy:** TESTING → INVALID occurs on the FIRST reset without confirmation, not after N failed boots. This matches MCUboot's test/confirm model: the application gets exactly one chance to prove itself healthy. If it fails to call `boot_set_confirmed()` before the next reset (any reset — watchdog, power cycle, crash), the bootloader immediately reverts to the fallback slot. This eliminates the N-boot unavailability window and prevents a crashing application from consuming multiple boot attempts.

---

## 8. DeviceID

```rust
/// Unique device identifier.
struct DeviceID {
    uid:             [u8; 16],   // Random 128-bit identifier (UUIDv4 format)
    manufacturer_id: u16,        // Manufacturer code
    model_id:        u16,        // Device model code
    hardware_rev:    u8,         // Hardware revision
    reserved:        [u8; 3],    // Must be zero
}
// Total: 24 bytes
```

---

## 9. KeyRef

```rust
/// Abstract reference to a cryptographic key.
/// Never contains key material — only identifies which key to use.
struct KeyRef {
    key_id:    u32,     // Key identifier
    key_type:  KeyType, // Type of key
    key_usage: u8,      // Bitmask of allowed operations
    reserved:  [u8; 2],
}

enum KeyType: u8 {
    ROOT         = 0x00,  // KR — Root key (trust anchor)
    DEVICE       = 0x01,  // KD — Device key (derived)
    IMAGE_VERIFY = 0x02,  // KI — Firmware verification key
    DEBUG_AUTH   = 0x03,  // KD_Debug — Debug authentication key
    BACKEND_AUTH = 0x04,  // Backend verification public key
    OTA_ENCRYPT  = 0x05,  // KO — OTA encryption key (X25519 ECDH)
}

// Key usage flags
const KEY_USAGE_VERIFY: u8 = 0x01;  // Can be used for signature verification
const KEY_USAGE_DERIVE: u8 = 0x02;  // Can be used as KDF input
const KEY_USAGE_AUTH:   u8 = 0x04;  // Can be used for authentication
```

---

## 10. AuthToken

```rust
/// Authentication token returned by backend after device authentication.
struct AuthToken {
    token_data:   [u8; 32],   // HMAC-SHA-256 token
    issued_at:    u64,        // Unix timestamp of issuance
    expires_at:   u64,        // Unix timestamp of expiry
    nonce:        [u8; 16],   // Random nonce from auth request
}
```

---

## 11. UpdateInfo

```rust
/// OTA update descriptor from backend.
struct UpdateInfo {
    update_available: bool,          // Whether an update exists
    firmware_version: u32,           // New firmware version
    image_size:      u32,            // Size in bytes
    image_hash:      [u8; 32],       // SHA-256 of full image
    signature:       SignatureBlock, // Backend signature over metadata
    download_url:    UrlString,      // Where to download the image (variable length)
    min_battery_pct: u8,             // Minimum battery percentage required (0 = no constraint)
    priority:        UpdatePriority, // Update priority level
}

enum UpdatePriority: u8 {
    CRITICAL = 0x00,  // Must install immediately (security fix)
    HIGH     = 0x01,  // Should install soon
    NORMAL   = 0x02,  // Install at next convenient time
    LOW      = 0x03,  // Optional update
}
```

---

## 12. ProvisioningData

```rust
/// Data exchanged during the provisioning process.
struct ProvisioningData {
    device_uid:       DeviceID,       // Generated unique device ID
    attestation_key:  PublicKey,      // Device attestation public key
    attestation_proof: [u8; 64],      // Signature(attestation_key, UID || metadata)
    root_key_ref:     KeyRef,         // Reference to injected root key
    device_key_ref:   KeyRef,         // Reference to derived device key
    firmware_version: u32,            // Version of provisioned firmware
    firmware_hash:    [u8; 32],       // Hash of provisioned firmware
    station_id:       [u8; 16],       // Provisioning station identifier
    operator_id:      [u8; 16],       // Operator identifier
    timestamp:        u64,            // Provisioning timestamp
}
```

---

## 13. TelemetryRecord

```rust
/// Device telemetry report sent to backend.
struct TelemetryRecord {
    device_uid:         DeviceID,    // Device identifier
    boot_count:         u32,         // Total boot count since provisioning
    boot_success_count: u32,         // Successful boots
    last_boot_error:    u32,         // Last boot error code (0 = no error)
    active_version:     u32,         // Currently active firmware version
    rollback_count:     u16,         // Number of OTA rollbacks
    tamper_events:      u16,         // Number of tamper events detected
    debug_access_count: u16,         // Debug access attempts
    uptime_seconds:     u32,         // Current uptime in seconds
    reserved:           [u8; 6],
}
```

---

## 14. DebugAuthChallenge

```rust
/// Debug authentication challenge-response data.
struct DebugAuthChallenge {
    challenge:       [u8; 32],  // Random challenge from device
    device_uid:      DeviceID,  // Device identifier
    access_level:    u8,        // Requested debug access level (1-2)
    reserved:        [u8; 3],
}

struct DebugAuthResponse {
    hmac:            [u8; 32],  // HMAC-SHA-256(KD_Debug, challenge || UID || access_level)
    access_level:    u8,        // Granted access level
    reserved:        [u8; 3],
}
```

---

## 15. CriticalMetadata

```rust
/// Critical metadata journal entry (128 bytes).
/// Stored in two-copy atomic journal at NOR 0x20_0000 + 0x20_1000.
/// Updated ONLY on OTA/slot transitions — NOT every boot.
/// Power-loss safe via two-copy atomic protocol with monotonic sequence numbers.
struct CriticalMetadata {
    sequence:          u32,       // Monotonic (odd=writing, even=committed)
    active_slot:       SlotID,    // Currently active (confirmed) slot
    boot_target:       SlotID,    // Slot to attempt boot from
    re_encrypt_state:  u8,        // 0x00=idle, 0x01=staged, 0x02=committed
    reserved:          u8,        // Must be zero
    slot_a_info:       ImageInfo, // Slot A metadata (52 B)
    slot_b_info:       ImageInfo, // Slot B metadata (52 B)
    max_allowed_version: u32,     // Anti-rollback version ceiling
    reserved2:         [u8; 8],   // Must be zero
    crc32:             u32,       // CRC-32 over bytes 0..124
}
// Total: 128 bytes
```

### 15.1 re_encrypt_state Values

| Value | Name | Meaning |
|-------|------|---------|
| 0x00 | RE_STATE_IDLE | No re-encrypt in progress |
| 0x01 | RE_STATE_STAGED | KD-encrypted data written to staging area, ready to commit |
| 0x02 | RE_STATE_COMMITTED | Staging data committed, old area pending erase |

---

## 16. FrequentRecord

```rust
/// Per-boot record in the frequent journal ring buffer (64 bytes).
/// Stored in ring buffer at NOR 0x20_2000 (4 KB sector, 64 records).
/// Appended on every successful boot — no sector erase until ring wraps.
/// Power-loss safe by design: losing a record only loses that boot's counters,
/// never affects critical slot state.
/// The pcr_value field provides a chained measurement for remote attestation
/// (measured boot, see §15.3). Each record extends the chain from the previous.
struct FrequentRecord {
    boot_seq:        u32,       // [0]  Monotonic boot counter (1, 2, 3, ...)
    prev_boot_phase: u8,        // [4]  0x00=did not reach EXECUTE, 0x01=reached EXECUTE, 0x02=confirmed
    flags:           u8,        // [5]  bit0=was_testing, bit1=was_fallback, bits2-7=reserved
    reserved:        [u8; 2],   // [6]  Must be zero
    last_iv_hash:    [u8; 24],  // [8]  Truncated SHA-256(IV_dev || KD[0..16]) for IV reuse detection
    pcr_value:       [u8; 32],  // [32] SHA-256 chained measurement (see §15.3)
}
// Total: 64 bytes. CRC-16 over bytes 0..62 stored at bytes [62..64].
```

### 16.1 prev_boot_phase Values

| Value | Name | Meaning |
|-------|------|---------|
| 0x00 | PHASE_NONE | Boot did not reach EXECUTE (crashed or failed verification) |
| 0x01 | PHASE_EXECUTE | Bootloader reached EXECUTE and jumped to application |
| 0x02 | PHASE_CONFIRMED | Application called boot_set_confirmed() (promoted TESTING→ACTIVE) |

### 16.2 flags Bit Definitions

| Bit | Name | Description |
|-----|------|-------------|
| 0 | FLAG_WAS_TESTING | This boot's slot was in TESTING state (image unconfirmed) |
| 1 | FLAG_WAS_FALLBACK | This boot used the fallback slot (after rollback) |
| 2-7 | Reserved | Must be zero |

### 16.3 Measured Boot PCR Chain

The `pcr_value` field implements a chained measurement log similar to TPM Platform Configuration Registers (PCRs). Each boot extends the chain from the previous boot's measurement, creating a tamper-evident log. A remote attestation verifier can validate the entire boot history by recomputing the chain.

```
PCR[0] = SHA-256("SBOP_MEASURED_BOOT_V1" || bootloader_version || device_uid)
PCR[n] = SHA-256(PCR[n-1] || image_hash[0..32] || slot || boot_phase)
```

Where:
- `bootloader_version`: u32 version of the bootloader itself
- `device_uid`: u8[16] unique device identifier (from DeviceID.uid)
- `image_hash`: u8[32] SHA-256 of the booted firmware image
- `slot`: SlotID of the booted image
- `boot_phase`: u8 boot result (0x01 = EXECUTE, 0xFF = FAILSAFE)

**Extend operation** (Phase 11 EXECUTE):

```
let prev = frequent_journal_read_last()
let prev_pcr = prev.valid ? prev.pcr_value : initial_pcr()
let new_pcr = sha256(prev_pcr || header.hash || slot as u8 || 0x01)
```

**Attestation verification** (remote verifier):

```
// Recompute the chain from the initial PCR
let expected = initial_pcr(device_uid)
for each measurement in measurement_log:
    expected = sha256(expected || measurement.image_hash || measurement.slot || measurement.boot_phase)
assert(expected == device.current_pcr)
```

**Tamper-evidence:** Any modification to a measurement record changes that record's pcr_value, which breaks the chain for all subsequent records. The verifier detects this immediately — the final PCR won't match the recomputed chain. There is no way to forge a valid chain without knowing all previous measurements and computing SHA-256 preimages (computationally infeasible).

---

## 17. ProgressRecord

```rust
/// OTA download progress checkpoint (64 bytes).
/// Stored in ring buffer at NOR 0x20_3000 (4 KB sector, 64 records).
/// Written every 32 KB during OTA download. Enables HTTP Range resume
/// after power loss or network interruption.
/// Cleared on successful download completion.
struct ProgressRecord {
    magic:          u32,       // 0x4F544150 ("OTAP") — record magic
    sequence:       u32,       // Chunk counter (0, 1, 2, ...)
    target_slot:    SlotID,    // SLOT_A or SLOT_B — which slot is being written
    flags:          u8,        // Reserved (must be 0)
    reserved:       [u8; 2],   // Must be zero
    bytes_written:  u32,       // Total bytes written so far
    total_size:     u32,       // Expected total image size (from UpdateInfo)
    expected_hash:  [u8; 32],  // SHA-256 of full image (from backend)
    running_hash:   [u8; 8],   // Truncated SHA-256 of data[0..bytes_written)
    crc32:          u32,       // CRC-32 over bytes 0..59
}
// Total: 64 bytes
```

### 17.1 Validation Rules

| Rule | Description |
|------|-------------|
| R-PROG-001 | `magic` must == 0x4F544150 |
| R-PROG-002 | `crc32` must match computed CRC-32 over bytes 0..59 |
| R-PROG-003 | On resume, `expected_hash` must match current UpdateInfo.image_hash |
| R-PROG-004 | On resume, stored data[0..bytes_written] SHA-256[0..8] must match `running_hash` |
| R-PROG-005 | `target_slot` must be SLOT_A or SLOT_B (not SLOT_INVALID) |

---

## 18. Serialization Rules

| Rule | Description |
| --- | --- |
| SR-001 | All structs serialize in field declaration order |
| SR-002 | Variable-length fields (arrays, strings) must have a preceding length field |
| SR-003 | Enum variants serialize as their declared discriminant value |
| SR-004 | Boolean fields serialize as u8 (0 = false, 1 = true) |
| SR-005 | Unknown enum discriminant values must be rejected (no silent default) |
| SR-006 | Forward compatibility: unknown bits in flags must be ignored (not rejected) |
| SR-007 | Backward compatibility: new struct versions append fields (never remove or reorder) |

---

## 19. Size Budgets

| Structure | Size | Notes |
| --- | --- | --- |
| ImageHeader | 80 bytes | Fixed (header_size = 80) |
| OTAImagePackage (header) | 44 bytes | Fixed OTA header (magic..reserved) |
| OTAImagePackage (GCM tag) | 16 bytes | Per-package |
| OTAImagePackage (signature) | 64 bytes | Ed25519 over plaintext |
| SignatureBlock (Ed25519) | 116 bytes | 1+1+2+4+32+72+4 |
| ImageInfo | 52 bytes | Fixed |
| CriticalMetadata | 128 bytes | Two copies = 256 B NOR |
| FrequentRecord | 64 bytes | 64 records per 4 KB sector. Includes 32 B PCR value for measured boot. |
| ProgressRecord | 64 bytes | 64 records per 4 KB sector |
| DeviceID | 24 bytes | Fixed |
| KeyRef | 8 bytes | Fixed |
| AuthToken | 56 bytes | Fixed |
| UpdateInfo | 108 bytes | + URL string |
| ProvisioningData | 188 bytes | + variable fields |
| TelemetryRecord | 62 bytes | Fixed |
| DebugAuthChallenge | 56 bytes | Fixed |
| DebugAuthResponse | 36 bytes | Fixed |
| **Total** | **1,134 bytes** | All fixed structures combined |

**Total static overhead per firmware image:** 196 bytes (ImageHeader 80 B + SignatureBlock 116 B)
**Total OTA transport overhead:** 124 bytes (OTAHeader 44 B + GCM tag 16 B + Ed25519 sig 64 B)

**Total metadata journal NOR usage:** 16 KB (Critical Copy A 4 KB + Critical Copy B 4 KB + Frequent ring 4 KB + OTA Progress 4 KB)
