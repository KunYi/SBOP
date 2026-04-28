# Signature Format Specification

**Document ID:** SUB-CRYPTO-SIG-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the exact encoding, structure, and verification rules for cryptographic signatures in SBOP. The signature format is part of the firmware image format and must be parsed correctly and safely under all inputs.

---

## 2. Signature Encoding

### 2.1 ECDSA P-256 Signature

```
Format: ASN.1 DER-encoded ECDSA-Sig-Value

SEQUENCE (0x30)
  ├── INTEGER r (0x02)
  │     ├── length (0x20 or 0x21 with leading zero)
  │     └── r value (32 bytes, big-endian)
  └── INTEGER s (0x02)
        ├── length (0x20 or 0x21 with leading zero)
        └── s value (32 bytes, big-endian)

Total: 70-72 bytes (typical), max 72 bytes
```

**Alternative (recommended):** Raw r||s format (64 bytes fixed) for simpler parsing and no ASN.1 attack surface. Selected by `encoding` field in SignatureBlock (0x00 = raw).

### 2.2 Ed25519 Signature

```
Format: Raw 64-byte fixed-length signature

R (32 bytes) || S (32 bytes)

Total: 64 bytes (always)
```

Per RFC 8032 §3.3. No encoding variation — always raw bytes.

### 2.3 SignatureBlock Structure

The authoritative SignatureBlock structure is defined in `Data_Structures.md` §4. This section describes signature-specific validation rules.

→ Full struct definition with field-level validation rules: `../../02_System_Design/Data_Structures.md` §4

---

## 3. Verification Rules

### 3.1 Mandatory Checks

```
verify_signature(data, sig_block, public_key) → Result:

  1. Validate sig_block.algorithm ∈ {0x01, 0x02}
     → If invalid: return ERR-BOOT-CRYPTO-002

  2. Validate sig_block.encoding ∈ {0x00, 0x01}
     → If invalid: return ERR-BOOT-CRYPTO-002

  3. Validate sig_block.sig_length matches algorithm:
     - ECDSA DER: 70 ≤ len ≤ 72
     - ECDSA raw: len = 64
     - Ed25519: len = 64
     → If invalid: return ERR-BOOT-CRYPTO-001

  4. Validate sig_block.crc32 matches computed CRC
     → If mismatch: return ERR-BOOT-CRYPTO-001

  5. Parse DER structure (if DER encoding):
     - Verify SEQUENCE tag (0x30)
     - Verify INTEGER r tag (0x02)
     - Verify INTEGER s tag (0x02)
     - Verify total length matches
     → If invalid: return ERR-BOOT-CRYPTO-001

  6. Verify r, s are in [1, n-1] (ECDSA)
     → If invalid: return ERR-BOOT-CRYPTO-001

  7. Verify cryptographic signature using public_key
     → If invalid: return ERR-BOOT-CRYPTO-001

  8. All checks constant-time; no early return
```

### 3.2 Parsing Constraints

| # | Constraint |
| --- | --- |
| 1 | No partial verification — all checks must pass |
| 2 | No relaxed parsing — DER must be strict (no extra bytes, no non-minimal encoding) |
| 3 | Signature encoding errors treated identically to cryptographic failures |
| 4 | Must not crash, overflow, or OOB read on malformed input |
| 5 | ASN.1 parser must be fuzz-tested per Fuzzing_Strategy.md |

---

## 4. Input to Signature

**What is signed:**

```
signature_input = SHA-256(image_header_bytes || image_payload_bytes)

ECDSA: sign(SHA-256(signature_input))
Ed25519: sign(SHA-512(signature_input))  // internal per Ed25519 spec
```

Signing the hash rather than the raw image allows the signer to operate on a fixed-size 32-byte input regardless of image size. The signing tool (`sbop-sign`) computes this hash and sends it to the HSM.

---

## 5. Public Key Distribution

| Key | Where Embedded | Format |
| --- | --- | --- |
| KI public (ECDSA) | Bootloader (Zone 1), compiled at build | 64 bytes: x||y (uncompressed) |
| KI public (Ed25519) | Bootloader (Zone 1), compiled at build | 32 bytes |
| KR commitment | Device identity, backend | 32 bytes: HMAC(KR, "COMMIT") |

Multiple KI public keys may be embedded to support key rotation (transition window). The `signer_id` field in SignatureBlock identifies which key signed this image.

---

## 6. References

| Document | Reference |
| --- | --- |
| Crypto Algorithms | `Crypto_Algorithms.md` |
| Data Structures | `../../02_System_Design/Data_Structures.md` |
| Image Format | `../../02_System_Design/Image_Format.md` |
| Fuzzing Strategy | `../../05_Verification/Fuzzing_Strategy.md` |
| Implementation Constraints | `Implementation_Constraints.md` |
