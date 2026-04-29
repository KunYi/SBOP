# Cryptographic Algorithms Specification

**Document ID:** SUB-CRYPTO-ALG-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines all cryptographic algorithms, parameters, and constraints used in SBOP. This is the normative reference for algorithm selection — any deviation must be justified in an Architecture Decision Record.

---

## 2. Digital Signature

### 2.1 Algorithm: Ed25519

| Parameter | Value |
| --- | --- |
| Algorithm | Ed25519 (Edwards-curve Digital Signature Algorithm) |
| Curve | Curve25519 |
| Hash | SHA-512 (internal) |
| Key size | 256-bit private, 256-bit public |
| Signature size | 64 bytes (fixed) |
| Standard | RFC 8032, NIST SP 800-186 (FIPS 186-5) |
| Security level | 128-bit (classical) |

**Usage:** Firmware signature verification on device. KI private key signs; device verifies with KI public key. Ed25519 is the single required signature algorithm — simpler than ECDSA, deterministic (no RNG needed for signing), fixed-size signatures, and included in FIPS 186-5.

**Rationale over ECDSA P-256:**
- Deterministic signing — no per-signature randomness needed; eliminates an entire class of implementation failures (ECDSA nonce reuse)
- Fixed 64-byte signatures — no ASN.1 DER parsing, no variable-length handling
- 256-bit public key vs. 512-bit for P-256 uncompressed
- Simpler implementation — fewer opportunities for side-channel leakage
- Faster verification
- FIPS 186-5 includes Ed25519 (§7) — no compliance gap

---

## 3. Hash Function

### 3.1 SHA-256

| Parameter | Value |
| --- | --- |
| Algorithm | SHA-256 (Secure Hash Algorithm, 256-bit) |
| Output size | 256 bits (32 bytes) |
| Block size | 512 bits (64 bytes) |
| Standard | FIPS 180-4 |
| Security level | 128-bit collision resistance (classical) |

**Usage:**
- Firmware image integrity verification
- HMAC key in HKDF
- Hash commitments (e.g., KR commitment)
- Image header HMAC

### 3.2 Prohibited Algorithms

The following MUST NOT be used for any security purpose:
- MD5, SHA-1 (collision attacks practical)
- SHA-256 truncated below 128 bits
- Custom or modified hash constructions

---

## 4. Key Derivation Function (KDF)

### 4.1 HKDF (HMAC-based Key Derivation Function)

| Parameter | Value |
| --- | --- |
| Algorithm | HKDF per RFC 5869 |
| Underlying hash | SHA-256 |
| Extract phase | HKDF-Extract(salt, IKM) → PRK |
| Expand phase | HKDF-Expand(PRK, info, L) → OKM |

**Usage in SBOP:**

```
KD = HKDF-Extract(salt=KR, IKM=UID)       → 256-bit PRK
KD_Auth = HKDF-Expand(PRK=KD, info="SBOP-AUTH-v1", L=32)
KD_Debug = HKDF-Expand(PRK=KD, info="SBOP-DEBUG-v1", L=32)
KD_Storage = HKDF-Expand(PRK=KD, info="SBOP-STORAGE-v1", L=32)
```

**HKDF info strings:**
- "SBOP-AUTH-v1" — OTA mutual TLS client certificate
- "SBOP-DEBUG-v1" — Debug challenge-response authentication
- "SBOP-STORAGE-v1" — Optional flash encryption key

→ See `Key_Derivation.md` for the complete derivation tree

---

## 5. Random Number Generation

### 5.1 TRNG Requirements

| Parameter | Requirement |
| --- | --- |
| Entropy source | Hardware TRNG (True Random Number Generator) |
| Min entropy | 128 bits per sample (NIST SP 800-90B evaluated) |
| Health tests | Continuous health tests (repetition count, adaptive proportion) |
| Standard | NIST SP 800-90A/B/C |

**Usage:**
- UID generation during provisioning (must be unique across all devices)
- Nonce generation for OTA authentication sessions

### 5.2 DRBG (if needed)

If TRNG throughput is insufficient, a DRBG may be seeded from TRNG:
- CTR_DRBG or Hash_DRBG per NIST SP 800-90A
- Reseeded at minimum every 2^20 requests
- Never used for key generation (TRNG direct only)

---

## 6. Constant-Time Requirements

All cryptographic operations on the verification path MUST be constant-time:

| Operation | Requirement | Measurement |
| --- | --- | --- |
| Ed25519 verify | Fixed-time implementation per RFC 8032 §5.1 | TIM-002 |
| SHA-256 | Fixed-time for security-critical usage | TIM-001 |
| Hash comparison | `constant_time_compare` — no early return | TIM-001 |
| Version comparison | Fixed-sequence, no short-circuit | FI-004 |

→ See `Implementation_Constraints.md` and `../../04_Security/Side_Channel_Countermeasures.md`

---

## 7. Algorithm Rationale

### 7.1 Why Ed25519?

- Simpler implementation — fewer opportunities for errors
- Deterministic signing — no per-signature randomness needed
- Faster verification (batched verification possible)
- No need for ASN.1 DER encoding
- Fixed 64-byte signature — predictable storage
- FIPS 186-5 compliant (included as Edwards-curve digital signature algorithm)
- Recommended by IETF (RFC 8032), NIST (SP 800-186), and IANA

### 7.2 Why SHA-256?

- 128-bit collision resistance (no practical collisions known)
- Hardware acceleration widely available
- Smaller state and output than SHA-512 (important for embedded)
- Required for all SBOP hashing operations

### 7.3 Why HKDF?

- Standardized (RFC 5869), widely reviewed
- Clean separation of extract and expand phases
- No known attacks on HMAC-SHA256 construction
- Domain separation via info parameter prevents key reuse

---

## 8. References

| Document | Reference |
| --- | --- |
| Implementation Constraints | `Implementation_Constraints.md` |
| Key Derivation | `Key_Derivation.md` |
| Signature Format | `Signature_Format.md` |
| Side-Channel Countermeasures | `../../04_Security/Side_Channel_Countermeasures.md` |
| Formal Verification | `../../04_Security/Formal_Verification.md` |
| Data Structures (ImageHeader) | `../../02_System_Design/Data_Structures.md` |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-002) |
