# SBOP Crypto Subsystem Specification

**Document ID:** SUB-CRYPTO-OV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the Crypto subsystem architecture, interfaces, and responsibilities. The Crypto subsystem provides all cryptographic primitives used by Boot, OTA, and Identity subsystems — it is the single point of cryptographic truth on the device.

---

## 2. Scope

**Includes:**
- Signature verification (Ed25519)
- Hash computation (SHA-256)
- Key derivation (HKDF-SHA256)
- Constant-time comparison for security-critical checks
- Random number generation health monitoring
- Key reference management (KeyRef opaque handles)

**Excludes:**
- Hardware-specific crypto accelerator drivers (HAL concern)
- Physical key storage implementation (secure element / TEE)
- Key material generation (manufacturing / HSM concern)
- Firmware signing (host-side toolchain)

---

## 3. Subsystem Architecture

```
┌─────────────────────────────────────────────┐
│              Crypto Subsystem                 │
│                                               │
│  ┌─────────────┐  ┌───────────────┐          │
│  │  Signature   │  │    Hash       │          │
│  │  Verifier    │  │   Engine      │          │
│  │  (Ed25519    │  │  (SHA-256)    │          │
│  │   Ed25519)   │  │               │          │
│  └──────┬───────┘  └───────┬───────┘          │
│         │                  │                  │
│  ┌──────▼──────────────────▼───────┐          │
│  │       Constant-Time Layer        │          │
│  │  (constant_time_compare,         │          │
│  │   fixed-sequence op dispatch)    │          │
│  └──────┬──────────────────────────┘          │
│         │                                      │
│  ┌──────▼───────┐  ┌──────────────┐          │
│  │  Key Deriver │  │  TRNG Health │          │
│  │  (HKDF)      │  │   Monitor    │          │
│  └──────┬───────┘  └──────────────┘          │
│         │                                      │
│  ┌──────▼────────────────────────────────┐    │
│  │         Key Reference Manager          │    │
│  │  (KeyRef handles, no raw key export)   │    │
│  └───────────────────────────────────────┘    │
│                                               │
│         ┌──────────────┐                      │
│         │  HAL (Crypto │                      │
│         │  Accelerator)│                      │
│         └──────────────┘                      │
└─────────────────────────────────────────────┘
```

---

## 4. Design Principles

| # | Principle | Implementation |
| --- | --- | --- |
| 1 | Standard algorithms only | Ed25519, SHA-256, HKDF — no custom crypto |
| 2 | Constant-time on all security paths | No data-dependent timing in verify, compare, derive |
| 3 | Keys never exposed | KeyRef opaque handles only; raw key material never returned |
| 4 | Deterministic verification | Same input → same output; no nonce/randomness in verification |
| 5 | Fail-safe on all errors | Any crypto error → verification failure (never undefined) |
| 6 | Hardware acceleration optional | Works without hardware crypto; faster with it |

---

## 5. Interface Summary

| Function | Consumer | Description |
| --- | --- | --- |
| `crypto_verify_signature(data, sig, key)` | Boot, OTA | Verify Ed25519 signature |
| `crypto_compute_hash(data)` | Boot, OTA | SHA-256 hash computation |
| `crypto_constant_time_compare(a, b)` | Boot | Constant-time memory comparison |
| `crypto_derive_key(prk, info, len)` | Identity | HKDF key derivation |
| `crypto_trng_health_check()` | Boot | Verify TRNG entropy source healthy |

→ Formal contracts in `../../03_Subsystem/Interface_Contracts.md` §4

---

## 6. Security Levels

| SL | Crypto Requirement | Verification |
| --- | --- | --- |
| SL 1 | Ed25519, SHA-256 | TEST-CRYPTO-001, TEST-CRYPTO-002 |
| SL 2 | + Constant-time hash comparison | TIM-001 |
| SL 3 | + Constant-time signature verification, masking | TIM-002, TVLA |
| SL 4 | + Hardware security module for key operations | Physical pentest, formal verification |

→ See `../../04_Security/CAL_SLT_Definitions.md` for SL capability details

---

## 7. Error Handling

All crypto errors follow the ERR-CRYPTO-* scheme defined in `Error_Code_Catalog.md`.

**Critical rule:** Errors from different failure modes MUST be indistinguishable in timing — no early return that reveals which check failed.

---

## 8. Implementation Guidance

| Platform | Recommendation |
| --- | --- |
| ARM Cortex-M with TrustZone | CryptoCell-312 or mbedTLS with PSA Crypto API |
| ARM Cortex-M (no TrustZone) | wolfSSL or mbedTLS (software only) |
| RISC-V | OpenTitan crypto IP or libsodium |
| ESP32 | Hardware crypto via ESP-IDF |
| No HW acceleration | TweetNaCl (Ed25519) or monocypher |

---

## 9. References

| Document | Reference |
| --- | --- |
| Crypto Algorithms | `Crypto_Algorithms.md` |
| Implementation Constraints | `Implementation_Constraints.md` |
| Key Derivation | `Key_Derivation.md` |
| Signature Format | `Signature_Format.md` |
| Interface Contracts | `../../03_Subsystem/Interface_Contracts.md` §4 |
| Side-Channel Countermeasures | `../../04_Security/Side_Channel_Countermeasures.md` |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §5 |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-002, ADR-003) |
