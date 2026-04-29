# Signature Format Specification

**Document ID:** SUB-CRYPTO-SIG-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines the exact encoding, structure, and verification rules for cryptographic signatures in SBOP. The signature format is part of the firmware image format and must be parsed correctly and safely under all inputs.

---

## 2. Signature Encoding

### 2.1 Ed25519 Signature

```
Format: Raw 64-byte fixed-length signature

R (32 bytes) || S (32 bytes)

Total: 64 bytes (always)
```

Per RFC 8032 §3.3. No encoding variation — always raw bytes.

### 2.2 SignatureBlock Structure

The authoritative SignatureBlock structure is defined in `Data_Structures.md` §4. This section describes signature-specific validation rules.

→ Full struct definition with field-level validation rules: `../../02_System_Design/Data_Structures.md` §4

---

## 3. Verification Rules

### 3.1 Mandatory Checks

```
verify_signature(data, sig_block, public_key) → Result:

  1. Validate sig_block.algorithm == 0x02 (Ed25519)
     → If invalid: return ERR-BOOT-CRYPTO-003

  2. Validate sig_block.sig_length == 64
     → If invalid: return ERR-BOOT-CRYPTO-001

  3. Validate sig_block.crc32 matches computed CRC
     → If mismatch: return ERR-BOOT-CRYPTO-001

  4. Verify cryptographic signature using public_key
     → If invalid: return ERR-BOOT-CRYPTO-001

  5. All checks constant-time; no early return
```

### 3.2 Parsing Constraints

| # | Constraint |
| --- | --- |
| 1 | No partial verification — all checks must pass |
| 2 | Signature encoding errors treated identically to cryptographic failures |
| 3 | Must not crash, overflow, or OOB read on malformed input |
| 4 | Parser must be fuzz-tested per Fuzzing_Strategy.md |

---

## 4. Input to Signature

**What is signed:**

```
signature_input = SHA-256(image_header_bytes || image_payload_bytes)

Ed25519: sign(SHA-512(signature_input))  // internal per Ed25519 spec
```

Signing the hash rather than the raw image allows the signer to operate on a fixed-size 32-byte input regardless of image size. The Ed25519 signing tool (`sbop-sign`) computes SHA-256 of the image, then uses Ed25519 (which internally applies SHA-512).

---

## 5. Public Key Distribution

| Key | Where Embedded | Format |
| --- | --- | --- |
| KI public (Ed25519) | Bootloader (Zone 1), compiled at build | 32 bytes |
| KR commitment | Device identity, backend | 32 bytes: HMAC(KR, "COMMIT") |

Multiple KI public keys may be embedded to support key rotation (transition window). The `key_index` field in SignatureBlock identifies which key signed this image.

---

## 6. References

| Document | Reference |
| --- | --- |
| Crypto Algorithms | `Crypto_Algorithms.md` |
| Data Structures | `../../02_System_Design/Data_Structures.md` |
| Image Format | `../../02_System_Design/Image_Format.md` |
| Fuzzing Strategy | `../../05_Verification/Fuzzing_Strategy.md` |
| Implementation Constraints | `Implementation_Constraints.md` |
