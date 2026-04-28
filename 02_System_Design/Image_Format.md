# SBOP Firmware Image Format Specification

**Document ID:** SYS-IFMT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

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
| 0x00   | magic            | 4 bytes  | Fixed identifier (e.g., 0x53424F50 "SBOP") |
| 0x04   | header_version   | 2 bytes  | Header format version                      |
| 0x06   | header_size      | 2 bytes  | Size of header                             |
| 0x08   | image_size       | 4 bytes  | Payload size in bytes                      |
| 0x0C   | firmware_version | 4 bytes  | Monotonic version number                   |
| 0x10   | flags            | 4 bytes  | Feature flags                              |
| 0x14   | hash             | 32 bytes | SHA-256 of payload                         |
| 0x34   | reserved         | variable | Future use (must be zero)                  |

---

## 6. Firmware Payload

* Raw firmware binary
* Size defined by image_size
* Must be contiguous

---

## 7. Signature Block

See `Data_Structures.md` §4 for the authoritative SignatureBlock structure definition.

Logical layout:

| Offset | Field          | Size     | Description          |
| ------ | -------------- | -------- | -------------------- |
| 0x00   | algorithm      | 1 byte   | Algorithm identifier |
| 0x01   | encoding       | 1 byte   | Encoding format      |
| 0x02   | sig_length     | 2 bytes  | Signature length     |
| 0x04   | reserved       | 4 bytes  | Must be zero         |
| 0x08   | signature      | 72 bytes | Signature data (padded) |
| 0x50   | signer_id      | 16 bytes | KI key identifier    |
| 0x60   | crc32          | 4 bytes  | CRC-32 over preceding fields |

Total block size: 100 bytes (fixed).

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
| 0   | FLAG_ENCRYPTED    | Payload is encrypted |
| 1   | FLAG_COMPRESSED   | Payload is compressed |
| 2   | FLAG_DELTA        | Image is a delta/patch update |
| 3-7 | Reserved          | Must be zero |
| 8   | FLAG_PQC_SIGNATURE| Signature uses post-quantum algorithm |
| 9-31| Reserved          | Must be zero |

→ See `Data_Structures.md` §3.1 for the authoritative flags definition.

---

## 11. Validation Rules

Boot must enforce:

* magic must match
* header_version supported
* hash matches payload
* signature valid
* firmware_version passes rollback policy

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
