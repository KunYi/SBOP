# SBOP Firmware Image Format Specification

**Document ID:** SYS-IFMT-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines the binary structure of firmware images used across the SBOP system.

This format is the single authoritative structure for:

* Boot verification
* OTA transfer
* Backend packaging
* Cryptographic validation

---

## 2. Design Principles

* Platform-independent binary layout
* Fixed header structure for deterministic parsing
* Explicit versioning and extensibility
* Alignment-safe and endian-defined

---

## 3. Byte Order

* All multi-byte fields SHALL use little-endian encoding

---

## 4. Overall Layout

+------------------------+
| Image Header           |
+------------------------+
| Firmware Payload       |
+------------------------+
| Signature Block        |
+------------------------+

---

## 5. Image Header Structure

| Offset | Field            | Size     | Description                                |
| ------ | ---------------- | -------- | ------------------------------------------ |
| 0x00   | magic            | 4 bytes  | Fixed identifier 0x53424F50 ("SBOP")       |
| 0x04   | header_version   | 2 bytes  | Header format version (2)                  |
| 0x06   | header_size      | 2 bytes  | Total header size in bytes (= 80)          |
| 0x08   | image_size       | 4 bytes  | Payload size in bytes                      |
| 0x0C   | firmware_version | 4 bytes  | Monotonic version number                   |
| 0x10   | flags            | 4 bytes  | Feature flags (see §10)                    |
| 0x14   | hash             | 32 bytes | SHA-256 of firmware payload                |
| 0x34   | enc_key_index    | 1 byte   | KO key pair index for OTA ECDH (0..3, 0xFF = unused) |
| 0x35   | reserved         | 3 bytes  | Must be zero                               |
| 0x38   | header_hmac      | 16 bytes | HMAC-SHA-256-128(KD, header[0..56]) — SPI bus integrity |
| 0x48   | reserved2        | 8 bytes  | Must be zero (future use)                  |

**Total header size: 80 bytes.**

→ See `Data_Structures.md` §3 for the authoritative ImageHeader structure definition.

---

## 6. Firmware Payload

* Raw firmware binary
* Size defined by image_size
* Must be contiguous

---

## 7. Signature Block

See `Data_Structures.md` §5 for the authoritative SignatureBlock structure definition.

Logical layout:

| Offset | Field          | Size     | Description          |
| ------ | -------------- | -------- | -------------------- |
| 0x00   | algorithm      | 1 byte   | Algorithm identifier (0x02 = Ed25519) |
| 0x01   | key_index      | 1 byte   | Key slot index (0-3, 0xFF = single-key legacy) |
| 0x02   | sig_length     | 2 bytes  | Signature length (64 for Ed25519) |
| 0x04   | reserved       | 4 bytes  | Must be zero         |
| 0x08   | public_key     | 32 bytes | Ed25519 public key (embedded in-band) |
| 0x28   | signature      | 72 bytes | Signature bytes (right-padded with zeros) |
| 0x70   | crc32          | 4 bytes  | CRC-32 over bytes 0x00..0x6F |

Total block size: 116 bytes (fixed).

---

## 8. Signature Coverage

Signature MUST be computed over:

* Image Header (excluding signature block)
* Firmware Payload

---

## 9. Hash Definition

* SHA-256 over firmware payload only
* Stored in header.hash

---

## 10. Flags Definition

| Bit | Name              | Description |
| --- | ----------------- | ----------- |
| 0   | FLAG_ENCRYPTED    | Payload is encrypted (KD_Storage at rest) |
| 1   | FLAG_COMPRESSED   | Payload is compressed |
| 2   | FLAG_DELTA        | Image is a delta/patch update |
| 3-7 | Reserved          | Must be zero |
| 8   | FLAG_PQC_SIGNATURE| Signature uses post-quantum algorithm |
| 9   | FLAG_OTA_PENDING  | First boot after OTA — payload is AES-256-GCM encrypted with K_s. Bootloader must ECDH decrypt → Ed25519 verify plaintext → KD_Storage re-encrypt. |
| 10-11 | Reserved       | Must be zero |
| 12-15 | enc_key_index (in-band) | OTA encryption key pair index. Redundant with header.enc_key_index field — cross-checked at boot. |
| 16-31 | Reserved       | Must be zero |

→ See `Data_Structures.md` §3.1 for the authoritative flags definition.

---

## 11. Validation Rules

Boot must enforce all rules from `Data_Structures.md` §3.2:

| Rule | Description |
| --- | --- |
| R-IMG-001 | `magic` must == 0x53424F50 |
| R-IMG-002 | `header_version` must be ≤ MAX_SUPPORTED_HEADER_VERSION |
| R-IMG-003 | `header_size` must be ≥ 80 |
| R-IMG-004 | `image_size` must be > 0 and ≤ MAX_IMAGE_SIZE |
| R-IMG-005 | `firmware_version` must be > 0 |
| R-IMG-006 | Reserved bytes must be zero |
| R-IMG-007 | SHA-256 of payload must match `hash` |
| R-IMG-008 | `header_hmac` must == HMAC-SHA-256(KD, header[0..56]) truncated to 128 bits |
| R-IMG-009 | If `enc_key_index` in flags bits 12-15 differs from `header.enc_key_index`, reject image |

→ See `Data_Structures.md` §3.2 for the authoritative validation rules.

---

## 12. Alignment Requirements

* Header aligned to 4 bytes
* Payload aligned to 4 bytes
* Signature block aligned to 4 bytes

---

## 13. Extensibility

* Reserved fields must be ignored if zero
* New header_version defines extensions

---

## 14. Error Handling

Any parsing or validation error:

→ MUST result in rejection

---

## 15. Security Considerations

* No partial parsing allowed
* No fallback parsing modes
* All fields must be validated before use
