# SBOP Serial Recovery Protocol (SRP)

**Document ID:** SUB-BOOT-SRP-001
**Version:** 1.0
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

This document defines the Serial Recovery Protocol (SRP) — a UART-based binary protocol for recovering bricked or FAILSAFE devices. It provides authenticated firmware upload, status query, and device control commands over a simple serial link. SRP is the SBOP equivalent of MCUboot's MCUmgr/SMP serial transport but adds mandatory cryptographic authentication before any recovery operation.

→ Boot flow integration: `Boot_Flow_Pseudocode.md` §4 (Recovery Boot)
→ Error codes: `../../02_System_Design/Error_Code_Catalog.md`
→ Debug auth key: `../../02_System_Design/Key_Hierarchy.md`

---

## 2. Transport

| Parameter | Value |
|-----------|-------|
| Physical interface | USART1: PA9 (TX), PA10 (RX) |
| Baud rate | 3 Mbps (115200 fallback for legacy tools) |
| Data format | 8N1 (8 data bits, no parity, 1 stop bit) |
| Flow control | None (XON/XOFF disabled). Host must wait for response before next frame. |
| Max payload | 2048 bytes per frame |
| Framing | Magic + length prefix + CRC-16. No SLIP/COBS — binary framing is self-delimiting. |

**Baud rate selection:** 3 Mbps is the practical maximum for STM32H750 USART1 at 96 MHz APB2 clock (oversampling × 16 requires 48 MHz, oversampling × 8 requires 24 MHz → 3 Mbps is supported in oversampling × 8 mode). 115200 fallback is supported for debug tools that can't reach 3 Mbps.

---

## 3. Frame Format

```
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────────┬──────┐
│ magic│opcode│flags │length│seq   │resv  │payload (variable)  │crc16 │
│ u16  │ u8   │ u8   │ u16  │ u8   │ u8   │ u8[length]         │ u16  │
│ 2 B  │ 1 B  │ 1 B  │ 2 B  │ 1 B  │ 1 B  │ 0–2048 B           │ 2 B  │
└──────┴──────┴──────┴──────┴──────┴──────┴─────────────────────┴──────┘
Total header: 8 bytes. Total frame: 10 + length bytes.
```

| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | magic | u16 | 0x5352 ("SR"). All frames start with this. |
| 2 | opcode | u8 | Command or response type (see §4) |
| 3 | flags | u8 | bit 0: is_response (1 = response, 0 = command). bit 1: is_error (response carries error). bit 2: more_fragments (more frames follow for this command). bits 3-7: reserved. |
| 4 | length | u16 | Payload length in bytes (0–2048). Little-endian. |
| 6 | sequence | u8 | Monotonic sequence number. Incremented per frame. Wraps at 255. |
| 7 | reserved | u8 | Must be zero. |
| 8 | payload | u8[length] | Variable-length payload. Opaque to framing layer. |
| 8+length | crc16 | u16 | CRC-16-CCITT (0x1021) over bytes 0..(8+length-1). Little-endian. |

### 3.1 Flag Definitions

| Bit | Name | Description |
|-----|------|-------------|
| 0 | FLAG_RESPONSE | Set on response frames. Clear on command frames. |
| 1 | FLAG_ERROR | Response indicates an error. Payload contains error code (u32 LE). |
| 2 | FLAG_MORE | More fragments follow. Used for large firmware uploads spanning multiple frames. |
| 3-7 | Reserved | Must be zero. Receiver must ignore unknown flags. |

### 3.2 Framing Recovery

The receiver synchronizes by scanning for the magic bytes 0x52 0x53 ("SR"). Once found:
1. Read opcode, flags, length, sequence, reserved (6 bytes)
2. Read `length` bytes of payload
3. Read 2 bytes of CRC-16
4. Verify CRC-16 over header + payload. If mismatch: discard frame, re-enter sync scan.
5. If `sequence` matches expected next sequence: accept frame. Else: discard, re-enter sync scan.

**Sync timeout:** If no valid frame received within 500 ms, the receiver resets its sync state and begins scanning for magic again. This prevents a stuck sync state from a corrupt length field.

---

## 4. Command Set

### 4.1 Command Summary

| Opcode | Name | Auth Required | Description |
|--------|------|---------------|-------------|
| 0x00 | PING | No | Heartbeat/liveness check. Response echoes payload. |
| 0x01 | AUTH_CHALLENGE | No | Begin authentication. Device returns random challenge. |
| 0x02 | AUTH_RESPONSE | No | Host provides HMAC response to challenge. |
| 0x10 | QUERY_STATUS | Yes | Get device identity, slot status, and error log summary. |
| 0x11 | UPLOAD_FIRMWARE | Yes | Upload firmware image to a specified slot. Fragmented. |
| 0x12 | ERASE_SLOT | Yes | Erase a slot. Requires explicit confirmation byte. |
| 0x13 | RESET_DEVICE | Yes | Trigger system reset. Requires explicit confirmation byte. |
| 0x14 | GET_ERROR_LOG | Yes | Retrieve the full error log. |
| 0x15 | GET_VERSION | No | Get protocol version and device info (non-sensitive only). |
| 0xFF | ERROR | — | Error response (device → host only). Payload: u32 error code. |

### 4.2 PING (0x00)

```
Command (Host → Device):
  payload: arbitrary bytes (0–64 B). Echoed in response.

Response (Device → Host):
  payload: echo of command payload.

Errors: None (PING always succeeds).
```

### 4.3 AUTH_CHALLENGE (0x01)

```
Command (Host → Device):
  payload: empty (length = 0).

Response (Device → Host):
  payload:
    [0..32]  challenge  : u8[32]  // Random 256-bit challenge from TRNG
    [32..56] device_uid : u8[24]  // DeviceID (24 B, see Data_Structures.md)

Errors:
  ERR-ID-IDENT-001: Device not provisioned (cannot authenticate)
```

### 4.4 AUTH_RESPONSE (0x02)

```
Command (Host → Device):
  payload:
    [0..32]   hmac         : u8[32]   // HMAC-SHA-256(KD_Debug, challenge || opcode)
    [33]      access_level : u8       // Requested access level (1 = recovery, 2 = debug)
    [34..36]  reserved     : u8[3]    // Must be zero

Response (Device → Host):
  payload:
    [0]  access_granted : u8     // 0x00 = denied, 0x01 = level 1, 0x02 = level 2
    [1]  auth_timeout   : u16    // Session timeout in seconds (LE)

Errors:
  ERR-ID-AUTH-001: HMAC verification failed (authentication rejected)

Auth session timeout: After successful authentication, the session remains valid
for `auth_timeout` seconds. All authenticated commands must complete within this
window. A new AUTH_CHALLENGE extends the session.

Anti-brute-force: After 3 consecutive failed AUTH_RESPONSE attempts, the device
enters a 5-second lockout before accepting new AUTH_CHALLENGE frames. After 10
cumulative failures, lockout increases to 60 seconds.
```

### 4.5 QUERY_STATUS (0x10)

```
Command (Host → Device):
  payload: empty.

Response (Device → Host):
  payload:
    [0..24]  device_id      : DeviceID (24 B)
    [24]     active_slot    : u8
    [25]     boot_target    : u8
    [26]     failsafe_reason: u32    // Error code that triggered FAILSAFE (0 if not in FAILSAFE)
    [30]     slot_a_status  : SlotStatus (u8)
    [31]     slot_a_version : u32
    [35]     slot_b_status  : SlotStatus (u8)
    [36]     slot_b_version : u32
    [40]     rollback_count : u16
    [42]     tamper_events  : u16
    [44]     boot_count     : u32    // From frequent journal (last record)
    [48..52] reserved       : u8[4]

Errors: None (always succeeds when authenticated).
```

### 4.6 UPLOAD_FIRMWARE (0x11)

```
Command (Host → Device):
  First frame (FLAG_MORE may be set on subsequent frames):
    [0]     target_slot   : u8       // SLOT_A or SLOT_B
    [1]     image_size    : u32      // Total image size in bytes (LE)
    [5]     image_hash    : u8[32]   // Expected SHA-256 of full image
    [37]    chunk_offset  : u32      // Byte offset of this chunk (LE)
    [41..N] chunk_data    : u8[]     // Firmware chunk (≤ 2000 B)

  Subsequent frames (FLAG_MORE set, chunk_offset increments accordingly):
    [0]     chunk_offset  : u32      // Byte offset (LE)
    [4..N]  chunk_data    : u8[]     // Firmware chunk (≤ 2044 B)

Response (per frame):
  ACK:
    [0]     status        : u8 = 0x00     // ACK
    [1]     bytes_written : u32           // Total bytes written so far (LE)
  NAK:
    [0]     status        : u8 = 0x01     // NAK
    [1]     error         : u32           // Error code (LE)

Upload flow:
  1. Host sends first UPLOAD_FIRMWARE frame with target_slot, image_size, image_hash.
  2. Device erases target slot, writes chunk at chunk_offset.
  3. Device responds with ACK + bytes_written.
  4. Host sends next frame with incremented chunk_offset.
  5. Repeat until all bytes transferred.
  6. Device verifies SHA-256(image_data) == image_hash.
  7. If hash matches: device writes slot metadata (status → VERIFIED).
     If hash mismatch: device returns NAK with ERR-OTA-DOWNLOAD-002.
  8. Host sends RESET_DEVICE to activate.

Constraints:
  - Max chunk size: 2044 bytes (2048 max payload minus 4 B header).
  - Chunks must be sent in order. Out-of-order chunks return NAK.
  - If the session expires mid-upload, the device preserves partial data
    (OTA progress journal). Host can re-authenticate and resume.
  - Concurrent uploads are not supported (single slot target).

Errors:
  ERR-OTA-DOWNLOAD-002: Hash mismatch after all data received.
  ERR-OTA-DOWNLOAD-003: Slot is ACTIVE (must erase inactive slot only).
  ERR-STOR-WRITE-001: Flash write failure.
  ERR-ID-AUTH-001: Session expired (re-authenticate before continuing).
```

### 4.7 ERASE_SLOT (0x12)

```
Command (Host → Device):
  payload:
    [0]  target_slot  : u8       // SLOT_A or SLOT_B
    [1]  confirm      : u8       // Must be 0xA5 (prevents accidental erase)

Response:
  ACK:
    [0]  status       : u8 = 0x00
  NAK:
    [0]  status       : u8 = 0x01
    [1]  error        : u32

Errors:
  ERR-OTA-INSTALL-003: Attempted to erase ACTIVE slot (rejected).
  ERR-STOR-ERASE-001: Flash erase failure.
```

### 4.8 RESET_DEVICE (0x13)

```
Command (Host → Device):
  payload:
    [0]  confirm      : u8       // Must be 0xA5

Response:
  ACK:
    [0]  status       : u8 = 0x00
    // Device resets within 100 ms of sending ACK.
    // Host should wait for RECOVERY_READY beacon before sending more commands.

Errors:
  None (reset always proceeds after ACK).
```

### 4.9 GET_ERROR_LOG (0x14)

```
Command (Host → Device):
  payload:
    [0]  start_index  : u16      // First error entry to retrieve (0 = most recent)
    [1]  max_entries  : u8       // Max entries to return (≤ 32)

Response:
  payload:
    [0]     total_entries : u16      // Total error entries in log
    [2]     entry_count   : u8       // Entries in this response
    [3..N]  entries       : ErrorEntry[]  // Error entry array

  ErrorEntry (12 bytes):
    [0]  error_code   : u32      // Full error ID (e.g., ERR-BOOT-CRYPTO-001)
    [4]  boot_phase   : u8       // Boot phase when error occurred
    [5]  slot         : u8       // Slot being verified (SLOT_A / SLOT_B / 0xFF)
    [6]  timestamp    : u32      // Boot sequence number (from frequent journal)
    [10] reserved     : u8[2]

Errors: None (always succeeds when authenticated).
```

### 4.10 GET_VERSION (0x15)

```
Command (Host → Device):
  payload: empty.

Response (Device → Host):
  payload:
    [0]     protocol_version : u8    // SRP protocol version (1)
    [1]     sbop_major       : u8    // SBOP major version
    [2]     sbop_minor       : u8
    [3]     sbop_patch       : u8
    [4..19] device_uid       : u8[16] // UUID only (not full DeviceID — no manufacturer/model)
    [20]    board_id         : u16    // Board identifier
    [22]    flash_size_mb    : u8     // NOR flash size in MB

Errors: None (always succeeds, no auth required).
```

---

## 5. Authentication Protocol

### 5.1 Key Derivation

The debug authentication key KD_Debug is derived from the device key KD:

```
KD_Debug = HKDF-SHA-256(
    IKM = KD,
    salt = "SBOP-DEBUG-AUTH-V1",
    info = device_uid || "DEBUG",
    length = 32
)
```

KD_Debug is **never stored** — it is derived on demand in the SVC handler. The derivation is constant-time with respect to KD.

### 5.2 Challenge-Response Flow

```
Device (FAILSAFE mode)                    Host (Recovery Tool)
─────────────────────────                ────────────────────
Beacon: RECOVERY_READY (periodic)  ──→
                                   ←──  AUTH_CHALLENGE
Generate challenge (TRNG, 32 B)
AUTH_CHALLENGE_RSP(challenge, UID)  ──→
                                         Compute:
                                           KD_Debug = derive(KD_master)
                                           HMAC = HMAC-SHA-256(KD_Debug, challenge || 0x02)
                                   ←──  AUTH_RESPONSE(HMAC, access_level=1)
Verify HMAC with local KD_Debug
If valid: grant session
AUTH_GRANTED(level=1, timeout=300)   ──→
                                   ←──  QUERY_STATUS
STATUS_RSP(...)                     ──→
                                   ←──  UPLOAD_FIRMWARE(SLOT_B, size, hash, data...)
Upload and verify...
                                   ←──  RESET_DEVICE(confirm=0xA5)
ACK                                 ──→
[Device resets]
```

### 5.3 Anti-Brute-Force

| Attempts (cumulative) | Lockout Duration |
|----------------------|------------------|
| 1–3 | 0 s (immediate retry) |
| 4–6 | 5 s |
| 7–9 | 15 s |
| 10+ | 60 s (and tamper_log_write on 10th failure) |

Lockout counter resets on successful authentication. Lockout is enforced in hardware: a DTCM-resident counter with RTC-based timer comparison. An attacker power-cycling between attempts resets the counter (DTCM is volatile) but the tamper_log entry at 10 failures persists in NOR.

---

## 6. FAILSAFE Beacon

When the bootloader enters FAILSAFE state (both slots dead), it broadcasts a periodic beacon on USART1 so a connected recovery tool can detect the device without polling.

```
Beacon frame (sent every 500 ms while in FAILSAFE, no authentication):
  magic   : 0x5352
  opcode  : 0xFE (RECOVERY_READY)
  flags   : 0x00
  length  : 8
  payload :
    [0..4]  failsafe_reason : u32      // Error code that triggered FAILSAFE
    [5]     active_slot     : u8       // Last attempted boot slot
    [6]     fallback_slot   : u8       // Fallback slot (0xFF if none)
    [7]     reserved        : u8

Beacon is rate-limited: one frame every 500 ms. The device sleeps between beacons
(WFI with USART RX interrupt enabled) to minimize power consumption.
```

**Beacon detection (host tool):** The recovery tool opens the serial port and listens for 0x5352 magic. Once detected, it reads the failsafe_reason and presents diagnostic information to the operator. The tool then initiates AUTH_CHALLENGE to begin recovery.

---

## 7. Protocol State Machine

```
                         ┌──────────┐
                         │  RESET   │
                         └────┬─────┘
                              │ Boot enters FAILSAFE
                              ▼
                         ┌──────────┐
              ┌─────────→│ BEACON   │←─────────────┐
              │          └────┬─────┘              │
              │               │ Host connects       │
              │               │ AUTH_CHALLENGE      │
              │               ▼                     │
              │          ┌──────────┐              │
              │  ┌──────→│  AUTH    │←──────┐      │
              │  │       └────┬─────┘       │      │
              │  │            │ Auth OK      │      │
              │  │            ▼              │      │
              │  │       ┌──────────┐       │      │
              │  │       │ COMMAND  │───────┘      │
              │  │       └────┬─────┘  Auth failed  │
              │  │            │                      │
              │  │            │ RESET_DEVICE         │
              │  │            ▼                      │
              │  │       ┌──────────┐              │
              │  └───────│  RESET   │              │
              │   timeout└──────────┘              │
              │          session timeout            │
              └────────────────────────────────────┘
```

**State descriptions:**

| State | Description |
|-------|-------------|
| BEACON | Device in FAILSAFE, broadcasting periodic RECOVERY_READY frames. Accepts PING, GET_VERSION, and AUTH_CHALLENGE. |
| AUTH | Authentication in progress. Accepts AUTH_RESPONSE only. Times out after 10 s of inactivity. |
| COMMAND | Authenticated session active. Accepts all authenticated commands. Times out after `auth_timeout` seconds (default 300 s). |

---

## 8. Host Tool (Reference Implementation)

A Python reference tool (`sbop-recovery`) provides:

```
Usage: sbop-recovery [options] <serial_port>

Commands:
  status              Show device status and slot info
  upload <file>       Upload firmware to inactive slot
  erase <slot>        Erase specified slot (A/B)
  reset               Reset device
  logs                Retrieve error log
  recover <file>      Full recovery: upload + reset

Options:
  --baud RATE         Baud rate (default: 3000000)
  --debug-key FILE    Path to KD_Debug derivation key file
  --timeout SEC       Command timeout (default: 10)
```

---

## 9. Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Unauthenticated firmware upload | All upload commands require successful AUTH_RESPONSE with valid HMAC |
| Brute-force KD_Debug | Exponential lockout after 3, 6, 10 failures. Tamper log entry at 10 failures. |
| Replay attack (captured AUTH_RESPONSE) | Each AUTH_CHALLENGE uses a fresh 256-bit TRNG challenge. Replay probability < 2^-256. |
| Session hijacking | Session is bound to the serial port. No network-facing attack surface. |
| USART pin glitching | Frame CRC-16 detects corrupted frames. CRC mismatch → frame discarded. |
| Information leak via GET_VERSION | Returns only non-sensitive data (protocol version, board ID, flash size). No key material. |
| Forced reset during upload | OTA progress journal survives reset. Upload can resume after re-authentication. |
| KD_Debug extraction from device | KD_Debug is derived on-demand via SVC, never stored. Derivation is constant-time. Extraction requires full KD compromise, at which point the device is already owned. |

---

## 10. MCUboot Comparison

| Feature | MCUboot (MCUmgr/SMP) | SBOP SRP |
|---------|---------------------|----------|
| Transport | Serial (SLIP-framed) or BLE | Serial (binary framed, no SLIP overhead) |
| Authentication | **None by default** (optional HMAC in recent versions) | **Mandatory** — HMAC-SHA-256 challenge-response using KD_Debug |
| Anti-brute-force | No built-in lockout | Exponential lockout with tamper logging |
| Firmware upload | SMP image upload (CBOR-encoded) | Simple binary framing with CRC-16 per frame |
| Resume support | Application-level | Built-in via OTA progress journal |
| Protocol overhead | CBOR encoding + SLIP framing (~10-20% overhead) | Binary header (8 B per frame, max 2048 B payload, ~0.4% overhead) |
| Multi-image | Yes (upgrade + confirmed image) | No (single bootloader stage) |
| Slot management | Explicit slot map in SMP | Simple SlotID (A/B) |

SBOP SRP trades MCUmgr's flexibility (CBOR, multi-image, multiple transports) for simplicity and mandatory security. The binary framing is optimized for the failure case where every byte counts (slow UART recovery links). Authentication is non-optional because a serial recovery interface without auth is equivalent to an unlocked debug port.

---

## 11. Error Codes

New error codes for SRP-specific failures:

| Error ID | Severity | Description |
|----------|----------|-------------|
| ERR-SRP-AUTH-001 | ERROR | Authentication required for this command |
| ERR-SRP-AUTH-002 | ERROR | Session expired — re-authenticate |
| ERR-SRP-AUTH-003 | ERROR | Brute-force lockout active — retry after delay |
| ERR-SRP-FRAME-001 | ERROR | Invalid frame (bad magic, CRC mismatch) |
| ERR-SRP-FRAME-002 | ERROR | Frame too large (length > 2048) |
| ERR-SRP-FRAME-003 | ERROR | Sequence mismatch (out-of-order frame) |
| ERR-SRP-CMD-001   | ERROR | Unknown opcode |
| ERR-SRP-CMD-002   | ERROR | Command not permitted in current state |

---

## 12. Integration Points

| Integration | Reference |
|-------------|-----------|
| Boot flow recovery entry | `Boot_Flow_Pseudocode.md` §4 (Recovery Boot) |
| FAILSAFE entry point | `Boot_Flow_Pseudocode.md` §4 (Fail-Safe Handler) |
| KD_Debug derivation | `../../02_System_Design/Key_Hierarchy.md` |
| Error code registry | `../../02_System_Design/Error_Code_Catalog.md` |
| Debug authentication | `../../04_Security/Secure_Debug_Architecture.md` |
| OTA progress journal | `../Update/OTA_Flow_Pseudocode.md` §9 |
