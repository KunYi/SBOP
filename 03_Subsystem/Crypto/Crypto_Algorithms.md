# Cryptographic Algorithms Specification

**Document ID:** SUB-CRYPTO-ALG-001
**Version:** 2.2
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

// OTA session key (per-release, from X25519 ECDH shared secret):
K_s = HKDF-Expand(PRK=SharedSecret, info="SBOP-OTA-v1" || FirmwareVersion, L=32)
```

Where `K_s` is the per-release OTA session key derived from the X25519 ECDH shared secret (see §5). HKDF domain separation via versioned info string prevents cross-release key reuse.

**HKDF info strings:**
- "SBOP-AUTH-v1" — OTA mutual TLS client certificate
- "SBOP-DEBUG-v1" — Debug challenge-response authentication
- "SBOP-STORAGE-v1" — Device-unique flash encryption key (secondary encryption at rest)
- "SBOP-OTA-v1" — OTA firmware encryption session key (X25519 ECDH → AES-256-GCM)
- "SBOP-OTA-NONCE-v1" — GCM nonce derivation from ECDH shared secret

→ See `Key_Derivation.md` for the complete derivation tree

---

## 5. Key Agreement

### 5.1 X25519 (ECDH over Curve25519)

| Parameter | Value |
| --- | --- |
| Algorithm | X25519 (Elliptic Curve Diffie-Hellman over Curve25519) |
| Curve | Curve25519 (Montgomery form) |
| Key size | 256-bit private, 256-bit public |
| Shared secret | 256-bit (32 bytes) |
| Standard | RFC 7748 |
| Security level | 128-bit (classical) |

**Usage in SBOP:** OTA firmware encryption key agreement. The Development Team generates an ephemeral X25519 key pair per release. The shared secret is computed as:

```
SharedSecret = X25519(Ephemeral_Private, KO_public)
```

Where `KO_public` is the device's long-term X25519 public key, burned in bootloader OTP. The shared secret is then passed through HKDF to derive the AES-256-GCM session key.

**Key points:**
- Ephemeral key pair generated per OTA release (forward secrecy per release)
- Same KO_public for all devices (operational simplicity — single OTA image)
- Static device key pair (KO) is per-device unique for secondary encryption binding
- The ephemeral public key is included in the OTA image header for device-side ECDH

**Constant-time requirement:** X25519 scalar multiplication MUST be implemented with fixed-window or Montgomery ladder — no data-dependent branches or array indices based on secret key material.

---

## 6. Symmetric Encryption

### 6.1 AES-256-GCM

| Parameter | Value |
| --- | --- |
| Algorithm | AES-256-GCM (Galois/Counter Mode) |
| Key size | 256 bits (32 bytes) |
| Nonce/IV | 96 bits (12 bytes) — see construction below |
| Authentication tag | 128 bits (16 bytes) |
| Standard | NIST SP 800-38D |
| Security level | 128-bit (classical) |

**Usage in SBOP:**

1. **OTA firmware encryption** (sign-then-encrypt): The plaintext firmware is first signed with Ed25519, then encrypted with AES-256-GCM using a key derived from X25519 ECDH. This provides confidentiality during OTA transmission (defense-in-depth beyond TLS).

2. **Secondary device-unique encryption** (KD_Storage): After OTA download and GCM decrypt, the firmware is re-encrypted with KD_Storage (AES-256-GCM) for at-rest protection. Each device has a unique KD_Storage, so the flash image differs per device.

**GCM nonce construction (OTA encryption):**

```
Nonce = HKDF-Expand(PRK=SharedSecret, info="SBOP-OTA-NONCE-v1", L=12)
```

The nonce is derived deterministically from the shared secret to avoid needing to transmit it. Since each OTA release uses a unique ephemeral key, the (key, nonce) pair is guaranteed unique per encryption.

**GCM nonce construction (KD_Storage re-encrypt):**

```
Nonce = random(12)  // TRNG, stored alongside encrypted firmware
```

Per-device re-encryption uses random nonces stored in the slot metadata. IV reuse detection in the frequent journal (see `Boot_Flow_Pseudocode.md` Phase 6) prevents nonce replay attacks.

**AAD (Additional Authenticated Data):**

```
AAD = firmware_version || device_class || timestamp
```

The AAD binds the ciphertext to firmware metadata — prevents an attacker from swapping ciphertexts between different firmware versions or device classes.

### 6.2 OTA Encryption Info Strings

| Info String | Purpose |
| --- | --- |
| `"SBOP-OTA-v1"` | HKDF info for OTA session key (K_s) derivation from ECDH shared secret |
| `"SBOP-OTA-NONCE-v1"` | HKDF info for GCM nonce derivation from ECDH shared secret |

---

## 7. Random Number Generation

### 7.1 TRNG Requirements

| Parameter | Requirement |
| --- | --- |
| Entropy source | Hardware TRNG (True Random Number Generator) |
| Min entropy | 128 bits per sample (NIST SP 800-90B evaluated) |
| Health tests | Continuous health tests (repetition count, adaptive proportion) |
| Standard | NIST SP 800-90A/B/C |

**Usage:**
- UID generation during provisioning (must be unique across all devices)
- Nonce generation for OTA authentication sessions

### 7.2 DRBG (if needed)

If TRNG throughput is insufficient, a DRBG may be seeded from TRNG:
- CTR_DRBG or Hash_DRBG per NIST SP 800-90A
- Reseeded at minimum every 2^20 requests
- Never used for key generation (TRNG direct only)

---

## 8. Constant-Time Requirements

All cryptographic operations on the verification path MUST be constant-time:

| Operation | Requirement | Measurement |
| --- | --- | --- |
| Ed25519 verify | Fixed-time implementation per RFC 8032 §5.1 | TIM-002 |
| X25519 scalar mul | Fixed-window / Montgomery ladder, no secret-dependent branches | TIM-002 |
| AES-256-GCM | Constant-time GHASH + CTR — no secret-dependent table lookups | TIM-002 |
| SHA-256 | Fixed-time for security-critical usage | TIM-001 |
| Hash comparison | `constant_time_compare` — no early return | TIM-001 |
| Version comparison | Fixed-sequence, no short-circuit | FI-004 |

→ See `Implementation_Constraints.md` and `../../04_Security/Side_Channel_Countermeasures.md`

---

## 9. Algorithm Rationale

### 9.1 Why Ed25519?

- Simpler implementation — fewer opportunities for errors
- Deterministic signing — no per-signature randomness needed
- Faster verification (batched verification possible)
- No need for ASN.1 DER encoding
- Fixed 64-byte signature — predictable storage
- FIPS 186-5 compliant (included as Edwards-curve digital signature algorithm)
- Recommended by IETF (RFC 8032), NIST (SP 800-186), and IANA

### 9.2 Why SHA-256?

- 128-bit collision resistance (no practical collisions known)
- Hardware acceleration widely available
- Smaller state and output than SHA-512 (important for embedded)
- Required for all SBOP hashing operations

### 9.3 Why X25519?

- Same curve as Ed25519 (Curve25519) — code reuse for field arithmetic, smaller combined footprint
- Ephemeral-static ECDH provides forward secrecy per OTA release
- 256-bit keys, 32-byte public keys — compact for embedded OTA image headers
- No point validation needed (RFC 7748 §5) — simpler and safer than NIST curves
- Widely deployed (TLS 1.3, Signal, WireGuard) — well-reviewed implementations available

### 9.4 Why AES-256-GCM?

- Authenticated encryption (encrypt + MAC in one pass) — prevents Chosen-Ciphertext Attacks
- GCM is widely accelerated in hardware (ARMv8 Crypto Extensions, AES-NI)
- 128-bit authentication tag provides strong integrity guarantees
- 96-bit nonce construction from HKDF avoids nonce reuse (fatal for GCM)
- NIST SP 800-38D standardized — compliance path for FIPS 140-2/140-3

### 9.5 Why HKDF?

- Standardized (RFC 5869), widely reviewed
- Clean separation of extract and expand phases
- No known attacks on HMAC-SHA256 construction
- Domain separation via info parameter prevents key reuse

---

## 10. References

| Document | Reference |
| --- | --- |
| Implementation Constraints | `Implementation_Constraints.md` |
| Key Derivation | `Key_Derivation.md` |
| Signature Format | `Signature_Format.md` |
| Side-Channel Countermeasures | `../../04_Security/Side_Channel_Countermeasures.md` |
| Formal Verification | `../../04_Security/Formal_Verification.md` |
| Data Structures (ImageHeader) | `../../02_System_Design/Data_Structures.md` |
| Key Hierarchy | `../../02_System_Design/Key_Hierarchy.md` |
| Boot Flow Pseudocode | `../Boot/Boot_Flow_Pseudocode.md` |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-002) |
