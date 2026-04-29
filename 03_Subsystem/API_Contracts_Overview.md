# SBOP API Contracts Overview

**Document ID:** SUB-API-OV-001
**Version:** 1.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document provides a consolidated summary of all SBOP subsystem API contracts. It serves as the entry point for implementers, listing every public interface, its contract category, timing constraints, and references to the detailed specification.

→ Detailed contract specifications: `Interface_Contracts.md`
→ Error code registry: `../02_System_Design/Error_Code_Catalog.md`
→ Data structure definitions: `../02_System_Design/Data_Structures.md`

---

## 2. Contract Categories

| Category | Symbol | Description | Verification |
| --- | --- | --- | --- |
| Functional | F | Correctness under valid inputs | Unit + integration test |
| Timing | T | Execution time bounded and constant where required | Measurement (TIM-xxx) |
| Safety | S | Precondition violation → defined error, not UB | Negative test (NEG-xxx) |
| Security | SEC | Constant-time, no side-channel, no early return | TVLA, timing measurement |

---

## 3. Subsystem Interface Summary

### 3.1 Boot Subsystem

| # | API | Signature | Category | Max Time | Ref |
| --- | --- | --- | --- | --- | --- |
| 1 | `boot_verify_image` | `(SlotID) → Result<ImageInfo, BootError>` | F, S | 300 ms (total boot) | §3.1 |
| 2 | `boot_get_image_info` | `(SlotID) → Result<ImageInfo, BootError>` | F | < 1 ms | §3.2 |
| 3 | `boot_set_pending_image` | `(SlotID) → Result<(), BootError>` | F | < 10 ms | §3.3 |
| 4 | `boot_enter_failsafe` | `(BootError) → Never` | S | < 1 ms to enter | §3.4 |
| 5 | `boot_lock` | `() → Result<(), BootError>` | F, S | < 1 ms | §3.5 |
| 6 | `boot_get_measurement_log` | `() → Result<FrequentRecord[], BootError>` | F | < 5 ms | §3.6 |

**State machine context:** Boot APIs are valid only during boot phases (INIT through EXECUTE). After `boot_lock`, no further boot-time modifications are possible.

**Error domain:** ERR-BOOT-CRYPTO-xxx, ERR-BOOT-PARSE-xxx, ERR-BOOT-STATE-xxx, ERR-BOOT-VERSION-xxx

→ Implementation: `Boot/Boot_Flow_Pseudocode.md`
→ State detail: `Boot/Boot_State_Detail.md`
→ Failure model: `Boot/Boot_Failure_Model.md`
→ Measured boot: `Boot/Boot_Flow_Pseudocode.md` §11.3

→ Implementation: `Boot/Boot_Flow_Pseudocode.md`
→ State detail: `Boot/Boot_State_Detail.md`
→ Failure model: `Boot/Boot_Failure_Model.md`

---

### 3.2 Crypto Subsystem

| # | API | Signature | Category | Max Time | Ref |
| --- | --- | --- | --- | --- | --- |
| 1 | `crypto_verify_signature` | `(&[u8], &[u8], u16, SigType, KeyRef) → Result<bool, CryptoError>` | F, T, SEC | 20 ms (Ed25519) | §4.1 |
| 2 | `crypto_compute_hash` | `(&[u8], HashAlgo) → Result<[u8; 32], CryptoError>` | F, T, SEC | 100 ms (1 MB @ 100 MHz) | §4.2 |
| 3 | `crypto_constant_time_compare` | `(&[u8], &[u8], usize) → bool` | T, SEC | O(len) cycles, no early return | §4.3 |
| 4 | `crypto_derive_key` | `(&[u8], &[u8], &[u8], u16) → Result<KeyMaterial, CryptoError>` | F, T, SEC | Constant-time wrt IKM | §4.4 |
| 5 | `crypto_ecdh` | `(KeyRef, &[u8; 32]) → Result<[u8; 32], CryptoError>` | F, T, SEC | Constant-time X25519, < 50 ms | §4.5 |
| 6 | `crypto_aead_decrypt` | `(&KeyMaterial, &[u8; 12], &[u8], &[u8], &[u8; 16]) → Result<Vec<u8>, CryptoError>` | F, T, SEC | Constant-time GHASH, all-or-nothing | §4.6 |

**Timing contract:** All crypto APIs must execute in constant time with respect to secret inputs. `crypto_verify_signature` and `crypto_constant_time_compare` must not leak whether the result is success or failure through timing.

**Key handle model:** Keys are accessed via opaque `KeyRef` handles. Raw key material is never exposed to callers. Key access is gated by `key_usage` permissions.

→ Implementation constraints: `Crypto/Implementation_Constraints.md`
→ Algorithm details: `Crypto/Crypto_Algorithms.md`
→ Signature format: `Crypto/Signature_Format.md`
→ Key derivation: `Crypto/Key_Derivation.md`

---

### 3.3 OTA Subsystem

| # | API | Signature | Category | Max Time | Ref |
| --- | --- | --- | --- | --- | --- |
| 1 | `ota_authenticate` | `(DeviceID, &[u8; 32]) → Result<AuthToken, OTAError>` | F | < 5 s (network) | §5.1 |
| 2 | `ota_download_image` | `(&UrlString, &[u8; 32], SlotID) → Result<(), OTAError>` | F, S | Variable (supports resume) | §5.2 |
| 3 | `ota_activate_pending` | `() → Result<(), OTAError>` | F, S | < 300 ms + reset | §5.3 |
| 4 | `ota_rollback` | `() → Result<(), OTAError>` | F, S | < 300 ms + reset | §5.4 |

**Error domain:** ERR-OTA-AUTH-xxx, ERR-OTA-DOWNLOAD-xxx, ERR-OTA-INSTALL-xxx, ERR-OTA-ACTIVATE-xxx, ERR-OTA-ROLL-xxx

**Resume support:** `ota_download_image` must support HTTP Range resume from interrupted downloads.

**Slot safety:** OTA always targets the inactive slot. Any attempt to write the active slot is ERR-OTA-INSTALL-003 and must be caught by callee.

→ Implementation: `Update/OTA_Flow_Pseudocode.md`
→ State machine: `Update/OTA_State_Machine.md`
→ Failure model: `Update/OTA_Failure_Model.md`
→ Recovery: `Update/OTA_Recovery.md`

---

### 3.4 Identity Subsystem

| # | API | Signature | Category | Max Time | Ref |
| --- | --- | --- | --- | --- | --- |
| 1 | `identity_get_device_id` | `() → Result<DeviceID, IdentityError>` | F | < 1 ms | §6.1 |
| 2 | `identity_enter_provisioning` | `(ProvisioningAuth) → Result<ProvisioningSession, IdentityError>` | F, S | < 5 s (station auth) | §6.2 |
| 3 | `identity_lock` | `(&[u8; 64]) → Result<(), IdentityError>` | F, S | < 100 ms (fuse write) | §6.3 |

**Lifecycle states:** UNINITIALIZED → PROVISIONING → REGISTERED → LOCKED

**Irreversibility:** Once LOCKED, identity cannot be modified. This is enforced by hardware (fuse or OTP).

**Error domain:** ERR-ID-PROV-xxx, ERR-ID-AUTH-xxx, ERR-ID-IDENT-xxx, ERR-ID-KEY-xxx, ERR-ID-ATST-xxx, ERR-ID-CLONE-xxx

→ Provisioning flow: `Identity/Provisioning_Flow.md`
→ Anti-cloning: `Identity/Anti_Cloning.md`
→ Failure model: `Identity/Identity_Failure_Model.md`

---

### 3.5 Backend Interfaces (Abstract)

| # | API | Signature | Category | Notes |
| --- | --- | --- | --- | --- |
| 1 | `backend_register_device` | `(DeviceID, &[u8; 64], u32, &[u8; 32]) → Result<RegistrationToken, BackendError>` | F | Idempotent registration |
| 2 | `backend_request_update` | `(DeviceID, u32, AuthToken) → Result<UpdateInfo, BackendError>` | F, S | Enforces deployment policy |

**Contract:** Backend interfaces are abstract — the concrete transport (HTTPS/TLS, MQTT, CoAP) is platform-defined. The contract specifies only the logical pre/post conditions.

---

### 3.6 Serial Recovery Protocol (SRP)

| # | Command | Opcode | Auth Req | Category | Notes |
| --- | --- | --- | --- | --- | --- |
| 1 | `PING` | 0x00 | No | F | Heartbeat/liveness. Echoes payload. |
| 2 | `AUTH_CHALLENGE` | 0x01 | No | F, SEC | Begin authentication. Returns 32 B challenge + device UID. |
| 3 | `AUTH_RESPONSE` | 0x02 | No | F, SEC | HMAC-SHA-256(KD_Debug, challenge \|\| 0x02). 3 failures → lockout. |
| 4 | `QUERY_STATUS` | 0x10 | Yes | F | Device identity, slot status, FAILSAFE reason. |
| 5 | `UPLOAD_FIRMWARE` | 0x11 | Yes | F, S | Fragmented firmware upload. 2044 B max chunk. Supports resume. |
| 6 | `ERASE_SLOT` | 0x12 | Yes | F, S | Erase a slot. Requires 0xA5 confirmation byte. |
| 7 | `RESET_DEVICE` | 0x13 | Yes | F | Trigger system reset. Requires 0xA5 confirmation byte. |
| 8 | `GET_ERROR_LOG` | 0x14 | Yes | F | Retrieve error log entries. |
| 9 | `GET_VERSION` | 0x15 | No | F | Protocol version, SBOP version, board ID. |

**Error domain:** ERR-SRP-AUTH-xxx, ERR-SRP-FRAME-xxx, ERR-SRP-CMD-xxx

→ Protocol specification: `Boot/Serial_Recovery_Protocol.md`
→ Recovery boot flow: `Boot/Boot_Flow_Pseudocode.md` §5
→ Debug auth key: `../02_System_Design/Key_Hierarchy.md`

---

## 4. Cross-Cutting Contracts

### 4.1 Timing Summary

| Operation | Budget | Constant-Time | Measurement |
| --- | --- | --- | --- |
| Full boot (INIT → EXECUTE) | < 300 ms | — | End-to-end timing |
| Ed25519 verify | < 20 ms | Yes | TIM-001 |
| X25519 ECDH | < 50 ms | Yes | TIM-002 |
| AES-256-GCM decrypt (1 MB) | < 100 ms | Yes (GHASH) | TIM-002 |
| SHA-256 (1 MB) | < 100 ms | Yes (fixed len) | TIM-002 |
| Constant-time compare | O(len) cycles | Yes | TIM-002 |
| Flash write (sector) | < 10 ms | — | Platform measurement |
| OTA authenticate | < 5 s | — | Network-dependent |
| OTA download | Variable | — | Bandwidth-dependent |

### 4.2 Error Code Ranges

| Subsystem | Range | Count |
| --- | --- | --- |
| Boot (CRYPTO + PARSE + STATE + VERSION) | ERR-BOOT-xxx-001..099 | 32 |
| Crypto (SIG + HASH + KDF + ECDH + AEAD + RNG) | ERR-CRYPTO-xxx-001..099 | 14 |
| OTA (AUTH + DOWNLOAD + INSTALL + ACTIVATE + ROLL) | ERR-OTA-xxx-001..099 | 12 |
| Identity (PROV + AUTH + IDENT + KEY + ATST + CLONE) | ERR-ID-xxx-001..099 | 14 |
| Storage | ERR-STOR-xxx-001..099 | 4 |
| Hardware | ERR-HW-xxx-001..099 | 7 |
| SRP (AUTH + FRAME + CMD) | ERR-SRP-xxx-001..099 | 8 |
| **Total** | | **91** |

→ Full catalog: `../02_System_Design/Error_Code_Catalog.md`

### 4.3 Key Handle Model

All cryptographic APIs use `KeyRef` opaque handles. Callers never access raw key bytes:

```
KeyRef → { key_type, key_usage, key_id, ... }
```

| Key | KeyRef Type | Usage | Accessible From |
| --- | --- | --- | --- |
| KI (public) | IMAGE_VERIFY | KEY_USAGE_VERIFY | Boot (Zone 1) |
| KD | DEVICE | KEY_USAGE_DERIVE | Crypto engine only |
| KD_OTA | DEVICE | KEY_USAGE_ENCRYPT + KEY_USAGE_DECRYPT | OTA library (SVC) |
| KD_Auth | DEVICE | KEY_USAGE_SIGN | Identity (SVC) |
| KD_Debug | DEVICE | KEY_USAGE_SIGN | Debug auth (SVC) |

→ Key hierarchy: `../02_System_Design/Key_Hierarchy.md`

---

## 5. Contract Compliance Matrix

| Rule | Applies To | Verification Method |
| --- | --- | --- |
| CMPL-001 | All preconditions checked by caller | Code review + negative tests |
| CMPL-002 | All postconditions guaranteed by callee | Unit tests per contract |
| CMPL-003 | Error conditions are exhaustive | Error injection testing |
| CMPL-004 | Timing contracts verified by measurement | TIM-001, TIM-002, TVLA |
| CMPL-005 | Side effects documented and bounded | Architecture review |

---

## 6. Contract Versioning

Contracts follow semantic versioning:

| Change | Version Bump | Example |
| --- | --- | --- |
| Breaking signature, pre/post condition, or error set change | Major | Adding required parameter |
| New optional field, new error code (non-breaking) | Minor | Adding optional timeout parameter |
| Documentation clarification only | Patch | Fixing ambiguous wording |

---

## 7. Quick Reference — All APIs

```
Boot:
  boot_verify_image(SlotID)             → Result<ImageInfo, BootError>
  boot_get_image_info(SlotID)           → Result<ImageInfo, BootError>
  boot_set_pending_image(SlotID)        → Result<(), BootError>
  boot_enter_failsafe(BootError)        → Never
  boot_lock()                           → Result<(), BootError>

Crypto:
  crypto_verify_signature(&[u8], &[u8], u16, SigType, KeyRef) → Result<bool, CryptoError>
  crypto_compute_hash(&[u8], HashAlgo)                         → Result<[u8; 32], CryptoError>
  crypto_constant_time_compare(&[u8], &[u8], usize)            → bool
  crypto_derive_key(&[u8], &[u8], &[u8], u16)                 → Result<KeyMaterial, CryptoError>
  crypto_ecdh(KeyRef, &[u8; 32])                               → Result<[u8; 32], CryptoError>
  crypto_aead_decrypt(&KeyMaterial, &[u8; 12], &[u8], &[u8], &[u8; 16]) → Result<Vec<u8>, CryptoError>

OTA:
  ota_authenticate(DeviceID, &[u8; 32])       → Result<AuthToken, OTAError>
  ota_download_image(&UrlString, &[u8; 32], SlotID) → Result<(), OTAError>
  ota_activate_pending()                      → Result<(), OTAError>
  ota_rollback()                              → Result<(), OTAError>

Identity:
  identity_get_device_id()                    → Result<DeviceID, IdentityError>
  identity_enter_provisioning(ProvisioningAuth) → Result<ProvisioningSession, IdentityError>
  identity_lock(&[u8; 64])                    → Result<(), IdentityError>

Backend (Abstract):
  backend_register_device(DeviceID, &[u8; 64], u32, &[u8; 32]) → Result<RegistrationToken, BackendError>
  backend_request_update(DeviceID, u32, AuthToken)              → Result<UpdateInfo, BackendError>
```

---

## 8. References

| Document | Reference |
| --- | --- |
| Interface Contracts (detailed) | `Interface_Contracts.md` |
| Error Code Catalog | `../02_System_Design/Error_Code_Catalog.md` |
| Data Structures | `../02_System_Design/Data_Structures.md` |
| Key Hierarchy | `../02_System_Design/Key_Hierarchy.md` |
| Boot Pseudocode | `Boot/Boot_Flow_Pseudocode.md` |
| OTA Pseudocode | `Update/OTA_Flow_Pseudocode.md` |
| Crypto Algorithms | `Crypto/Crypto_Algorithms.md` |
| Implementation Constraints | `Crypto/Implementation_Constraints.md` |
| Side-Channel Countermeasures | `../04_Security/Side_Channel_Countermeasures.md` |
