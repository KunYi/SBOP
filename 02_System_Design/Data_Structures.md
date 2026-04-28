# SBOP Data Structure Definitions

**Document ID:** SYS-DS-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

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
/// Total size: 64 bytes (excluding reserved extension)
struct ImageHeader {
    magic:            u32,      // 0x53424F50 ("SBOP")
    header_version:   u16,      // Header format version (1)
    header_size:      u16,      // Total header size in bytes (≥ 64)
    image_size:       u32,      // Payload size in bytes
    firmware_version: u32,      // Monotonic version number
    flags:            u32,      // Feature flags (see Section 3.1)
    hash:             [u8; 32], // SHA-256 of firmware payload
    reserved:         [u8; 12], // Must be zero (future use)
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
| 9-31 | Reserved | Must be zero |

### 3.2 Validation Rules

| Rule | Description |
| --- | --- |
| R-IMG-001 | `magic` must == 0x53424F50 |
| R-IMG-002 | `header_version` must be ≤ MAX_SUPPORTED_HEADER_VERSION |
| R-IMG-003 | `header_size` must be ≥ 64 |
| R-IMG-004 | `image_size` must be > 0 and ≤ MAX_IMAGE_SIZE |
| R-IMG-005 | `firmware_version` must be > 0 (version 0 is invalid) |
| R-IMG-006 | Reserved bytes must be zero |
| R-IMG-007 | SHA-256 of payload must match `hash` |

---

## 4. SignatureBlock

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

### 4.1 Algorithm Values

| Value | Algorithm | Signature Size |
| --- | --- | --- |
| 0x02 | Ed25519 | 64 bytes |

### 4.2 Key Index Values

| Value | Meaning |
| --- | --- |
| 0x00 | Key slot 0 (primary signing key) |
| 0x01 | Key slot 1 (secondary / rotation) |
| 0x02 | Key slot 2 (backup) |
| 0x03 | Key slot 3 (backup) |
| 0xFF | Single-key legacy mode (public key baked into bootloader .rodata) |

### 4.3 Validation Rules

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

## 5. ImageInfo

```rust
/// Metadata about a firmware image in a slot.
/// Returned by boot subsystem queries.
struct ImageInfo {
    slot:             SlotID,    // Which slot this image is in
    status:           SlotStatus,// Current state of the slot
    firmware_version: u32,       // Version of firmware in this slot
    image_hash:       [u8; 32],  // SHA-256 of payload
    flags:            u32,       // Feature flags from header
    build_timestamp:  u64,       // Unix timestamp of build (optional)
    boot_count:       u32,       // Number of times this image has booted
}
```

---

## 6. SlotID and SlotStatus

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
    ACTIVE      = 0x03,  // Image is the active running firmware
    FALLBACK    = 0x04,  // Image is active but previous version (after rollback)
    INVALID     = 0x05,  // Image failed verification
    CORRUPT     = 0x06,  // Storage corruption detected in this slot
}
```

### 6.1 Slot State Transitions

```
EMPTY ──(write)──> PENDING
PENDING ──(verify OK)──> ACTIVE
PENDING ──(verify fail)──> INVALID
ACTIVE ──(new update to other slot)──> FALLBACK
ACTIVE ──(corruption detected)──> CORRUPT
FALLBACK ──(replaced by new update)──> PENDING
INVALID ──(new write)──> PENDING
CORRUPT ──(reformat)──> EMPTY
```

---

## 7. DeviceID

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

## 8. KeyRef

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
}

// Key usage flags
const KEY_USAGE_VERIFY: u8 = 0x01;  // Can be used for signature verification
const KEY_USAGE_DERIVE: u8 = 0x02;  // Can be used as KDF input
const KEY_USAGE_AUTH:   u8 = 0x04;  // Can be used for authentication
```

---

## 9. AuthToken

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

## 10. UpdateInfo

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

## 11. ProvisioningData

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

## 12. TelemetryRecord

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

## 13. DebugAuthChallenge

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

## 14. Serialization Rules

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

## 15. Size Budgets

| Structure | Size | Notes |
| --- | --- | --- |
| ImageHeader | 64 bytes | Fixed |
| SignatureBlock (Ed25519) | 116 bytes | 1+1+2+4+32+72+4 (algorithm + key_index + sig_length + reserved + public_key + signature + crc32) |
| ImageInfo | 52 bytes | + variable fields |
| DeviceID | 24 bytes | Fixed |
| KeyRef | 8 bytes | Fixed |
| AuthToken | 56 bytes | Fixed |
| UpdateInfo | 108 bytes | + URL string |
| ProvisioningData | 188 bytes | + variable fields |
| TelemetryRecord | 62 bytes | Fixed |
| DebugAuthChallenge | 56 bytes | Fixed |
| DebugAuthResponse | 36 bytes | Fixed |

**Total static overhead per firmware image:** ~180 bytes (ImageHeader + SignatureBlock)
