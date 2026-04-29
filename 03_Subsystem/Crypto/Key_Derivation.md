# Key Derivation Specification

**Document ID:** SUB-CRYPTO-KDF-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines the complete key derivation scheme for SBOP. All device keys are derived from the Root Key (KR) using HKDF (RFC 5869) with domain-separated info strings. Keys are never generated independently — they chain back to KR.

---

## 2. Key Hierarchy (Complete)

```
KR (Root Key) ─── 256-bit symmetric key, stored in HSM
 │
 ├── KD = HKDF-Extract(salt=KR, IKM=UID)
 │    │    256-bit Device Key, unique per device
 │    │    Stored in secure element / TEE
 │    │
 │    ├── KD_Auth = HKDF-Expand(KD, "SBOP-AUTH-v1", 32)
 │    │    │   OTA mutual TLS client certificate key
 │    │    │
 │    ├── KD_Debug = HKDF-Expand(KD, "SBOP-DEBUG-v1", 32)
 │    │    │   Debug challenge-response authentication key
 │    │    │
 │    └── KD_Storage = HKDF-Expand(KD, "SBOP-STORAGE-v1", 32)
 │         │   Optional flash encryption key
 │
 ├── KO (OTA Encryption Key) ─── X25519 key pair
 │    │                          Generated in HSM, private key never exported
 │    │                          Same KO_public burned in all device bootloaders
 │    │
 │    └── K_s (OTA Session Key) ─── AES-256-GCM key, per-release
 │         │                        K_s = HKDF-Expand(SharedSecret, "SBOP-OTA-v1" || Version, 32)
 │         │                        SharedSecret = X25519(Ephemeral_Private, KO_public)
 │         │                        Ephemeral key pair generated fresh per OTA release
 │
 └── KI (Image Signing Key) ─── Ed25519 key pair
      │                          Generated in HSM, private key never exported
      │
      └── KI_public ─── Embedded in bootloader at build time
                         Published in backend firmware registry
```

### 2.1 OTA Session Key (K_s) Derivation

The OTA session key K_s is derived per-release from an X25519 ECDH shared secret:

```
Ephemeral_Keypair = X25519_KeyGen()       // Fresh per OTA release (Development Team)
SharedSecret      = X25519(Ephemeral_Private, KO_public)
K_s               = HKDF-Expand(PRK=SharedSecret, info="SBOP-OTA-v1" || FirmwareVersion, L=32)
GCM_Nonce         = HKDF-Expand(PRK=SharedSecret, info="SBOP-OTA-NONCE-v1", L=12)
```

**Purpose:** Provides per-release firmware confidentiality during OTA transmission. The ephemeral key pair ensures forward secrecy — compromise of one release's K_s does not compromise other releases. The ephemeral public key is embedded in the OTA image header so the device can recompute the same shared secret using its KO_public (the static key burned in OTP).

**Sign-then-encrypt ordering:**
```
1. Sign plaintext firmware with Ed25519 (KI private key)
2. Encrypt signed firmware with AES-256-GCM using K_s
3. Assemble OTA image package (Header || Ephemeral_PubKey || Ciphertext || GCM_Tag || Signature)
```

The Ed25519 signature covers the plaintext firmware (not the ciphertext), so the signature can be verified before decryption without exposing plaintext to the OTA layer. The bootloader verifies the signature first, then decrypts.

---

## 3. Derivation Functions

### 3.1 Device Key (KD) Derivation

```
KD = HKDF-SHA256(
    salt  = KR,          // 256-bit Root Key (from HSM)
    IKM   = UID,         // 128-bit Unique Device ID (from TRNG)
    info  = "",          // empty info for KD itself
    L     = 32           // 256-bit output
)
```

**Purpose:** Binds each device to KR. KD is unique per device because UID is unique. The backend can re-derive KD for any device (given KR and UID) for device authentication verification. KR is never sent to the device — only KD is injected.

### 3.2 Sub-Key Derivation

All sub-keys are derived from KD using HKDF-Expand with domain-separated info strings:

```
KD_Auth = HKDF-SHA256(KD, info="SBOP-AUTH-v1", L=32)
KD_Debug = HKDF-SHA256(KD, info="SBOP-DEBUG-v1", L=32)
KD_Storage = HKDF-SHA256(KD, info="SBOP-STORAGE-v1", L=32)
```

**Domain separation:** The info prefix "SBOP-*" prevents accidental key reuse with other protocols. The "-v1" suffix enables future derivation algorithm upgrades.

---

## 4. Key Usage Matrix

| Key | Purpose | Derivation | Location | Access |
| --- | --- | --- | --- | --- |
| KR | Root of trust | HSM TRNG (provisioning) | HSM only | Quorum only |
| KD | Device identity | HKDF(KR, UID) | Secure element / TEE | KeyRef handle |
| KD_Auth | OTA auth (mTLS) | HKDF(KD, "SBOP-AUTH-v1") | Secure element | KeyRef handle |
| KD_Debug | Debug auth | HKDF(KD, "SBOP-DEBUG-v1") | Secure element | KeyRef handle |
| KD_Storage | Flash encryption (at rest) | HKDF(KD, "SBOP-STORAGE-v1") | Secure element | KeyRef handle |
| KO (private) | OTA ECDH key agreement | HSM TRNG (ceremony) | HSM only | Quorum + ceremony |
| KO (public) | OTA ECDH key agreement | KO key pair | Bootloader OTP | Device-side X25519 |
| K_s | OTA firmware encryption | HKDF(SharedSecret, "SBOP-OTA-v1") | RAM only (ephemeral) | Derived per-release, never stored |
| KI (private) | Firmware signing | HSM TRNG (ceremony) | HSM only | Quorum + ceremony |
| KI (public) | Firmware verification | KI key pair | Bootloader, backend | Public read |

---

## 5. Derivation Properties

| Property | Description |
| --- | --- |
| Deterministic | Same KR + UID → same KD every time (backend can re-derive) |
| Device-unique | Different UID → different KD (cryptographically independent) |
| Domain-separated | Different info → different sub-key (no cross-use) |
| Non-invertible | Given KD_Auth, cannot recover KD or KR (HKDF security property) |
| Verifiable | Backend can compute KD = HKDF(KR, UID) and verify device proofs |

---

## 6. Constraints

| # | Constraint |
| --- | --- |
| 1 | KR MUST never leave HSM (derive KD inside HSM, export KD only) |
| 2 | KD derivation MUST use TRNG UID (not fixed serial number) |
| 3 | Sub-keys MUST use domain-separated info strings (no empty info for sub-keys) |
| 4 | Derived keys SHOULD NOT be stored if they can be re-derived on demand |
| 5 | All derivation MUST be constant-time with respect to input key material |
| 6 | Backend MUST apply rate limiting to KD derivation requests |
| 7 | Backend MUST NOT expose raw KD outside HSM boundary |
| 8 | KO (private) MUST never leave HSM — ECDH operations performed inside HSM |
| 9 | Ephemeral X25519 key pair MUST be generated fresh per OTA release — never reused |
| 10 | K_s MUST be zeroized from RAM immediately after use (decrypt + re-encrypt complete) |

---

## 7. Derivation During Provisioning

```
Factory station:
  1. Device generates UID (TRNG) → reports to station
  2. Station sends UID to HSM (via backend or local HSM)
  3. HSM computes KD = HKDF(KR, UID)
  4. HSM returns KD to station (encrypted channel)
  5. Station injects KD into device secure element
  6. Device verifies KD: derives KD_Auth, sends proof to backend
  7. Station zeroizes KD from memory after injection
```

→ See `../../04_Security/Manufacturing_Security.md` §4.2 for the complete provisioning protocol

---

## 8. References

| Document | Reference |
| --- | --- |
| Crypto Algorithms | `Crypto_Algorithms.md` §4 |
| Key Hierarchy (System Design) | `../../02_System_Design/Key_Hierarchy.md` |
| Key Management (Operations) | `../../06_Operations/Key_Management.md` |
| Manufacturing Security | `../../04_Security/Manufacturing_Security.md` |
| Interface Contracts | `../../03_Subsystem/Interface_Contracts.md` §4, §8 |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-007, ADR-010) |
