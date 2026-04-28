# SBOP OTA Subsystem Specification

**Document ID:** SUB-OTA-OV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the OTA (Over-the-Air) subsystem responsible for secure and reliable firmware updates. The OTA subsystem orchestrates download, verification, staging, activation, and recovery of firmware images.

---

## 2. Scope

**Includes:**
- Backend authentication (mutual TLS)
- Firmware download and integrity verification
- Pre-install validation (device-side signature verification)
- Image staging to inactive slot
- Activation handoff to Boot subsystem
- Rollback and recovery

**Excludes:**
- Firmware execution and boot-time verification (Boot subsystem)
- Key provisioning and device identity creation (Identity subsystem)
- Firmware build and signing (Build pipeline / Operations)

---

## 3. Subsystem Architecture

```
┌──────────────────────────────────────────────────────┐
│                   OTA Subsystem                        │
│                                                        │
│  ┌───────────┐   ┌───────────┐   ┌────────────────┐  │
│  │ Backend   │   │ Download  │   │ Image          │  │
│  │ Auth      │──→│ Manager   │──→│ Verifier       │  │
│  │ (mTLS)    │   │           │   │                │  │
│  └───────────┘   └───────────┘   └───────┬────────┘  │
│                                          │             │
│  ┌───────────────────────────────────────┘             │
│  ▼                                                      │
│  ┌───────────┐   ┌───────────┐   ┌────────────────┐   │
│  │ Slot      │   │ Activation│   │ Rollback       │   │
│  │ Writer    │──→│ Handler   │──→│ Manager        │   │
│  └───────────┘   └───────────┘   └────────────────┘   │
│                                                        │
│  ┌────────────────────────────────────────────────┐   │
│  │ Telemetry Reporter (status, progress, errors)   │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
└──────────────────────────────────────────────────────┘
```

---

## 4. OTA State Machine

| State | Description | Next States |
|-------|-------------|-------------|
| IDLE | No update in progress | CHECK, ERROR |
| CHECK | Polling backend for available update | DOWNLOAD, IDLE |
| DOWNLOAD | Receiving firmware image | VERIFY, DOWNLOAD_ERROR |
| VERIFY | Device-side signature + hash check | STAGE, VERIFY_ERROR |
| STAGE | Writing image to inactive slot | ACTIVATE, STAGE_ERROR |
| ACTIVATE | Requesting boot switch to new image | IDLE, ROLLBACK |
| ROLLBACK | Reverting to previous valid image | IDLE |
| ERROR | Terminal error, awaiting recovery command | IDLE (on recovery) |

---

## 5. Transport & Authentication

| Requirement | Mechanism |
|-------------|-----------|
| Confidentiality | TLS 1.3 (AEAD ciphers only) |
| Server authentication | TLS server certificate (CA-issued, no self-signed) |
| Client authentication | Mutual TLS with KD_Auth-derived client certificate |
| Replay protection | TLS 1.3 inherent + session tokens (time-limited) |
| Download resume | Range requests supported (interruption recovery) |

---

## 6. Key Interfaces

| Interface | Function | Direction | Error Codes |
|-----------|----------|-----------|-------------|
| Crypto | `verify_signature(image, sig, pubkey)` | Call | ERR-OTA-SIG-001 |
| Crypto | `compute_hash(image)` | Call | ERR-OTA-HASH-001 |
| Identity | `get_device_certificate()` | Call | ERR-OTA-AUTH-001 |
| Identity | `get_device_uid()` | Call | ERR-OTA-ID-001 |
| Boot | `get_slot_info(slot)` | Call | ERR-OTA-SLOT-001 |
| Boot | `request_activation(slot)` | Call | ERR-OTA-ACT-001 |
| Backend | `check_update(uid, version)` | mTLS | ERR-OTA-NET-001 |
| Backend | `download_firmware(url, range)` | mTLS | ERR-OTA-NET-002 |
| Backend | `report_status(uid, state)` | mTLS | ERR-OTA-NET-003 |
| Storage | `write_slot(slot, offset, data)` | Call | ERR-OTA-STOR-001 |

---

## 7. Safety Guarantees

| # | Guarantee | Mechanism |
|---|-----------|-----------|
| 1 | Running image never overwritten | Write only to inactive slot |
| 2 | Interrupted update does not corrupt active image | Dual-slot isolation, atomic slot marking |
| 3 | Unverified image never booted | Device-side signature + hash verification before activation |
| 4 | Rollback always available | Previous valid image preserved until new image verified |
| 5 | Backend compromise does not enable device compromise | Device independently re-verifies all firmware |
| 6 | Rollback protection maintained across OTA | OTP counter checked at boot, not during OTA |

---

## 8. Recovery Strategy

| Scenario | Recovery |
|----------|----------|
| Download interrupted | Resume via Range request; timeout triggers retry |
| Verification failure | Discard downloaded image; re-download |
| Stage failure | Mark slot invalid; retry from CHECK |
| Activation failure | Boot subsystem detects invalid slot; falls back to other slot |
| New image crashes after boot | Watchdog triggers reset; Boot selects previous slot |

---

## 9. References

| Document | Reference |
|----------|-----------|
| OTA Flow Pseudocode | `OTA_Flow_Pseudocode.md` |
| OTA Interface | `OTA_Interface.md` |
| OTA Failure Model | `OTA_Failure_Model.md` |
| OTA Deployment | `../../04_Security/OTA_Deployment.md` |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` |
| Trust Model | `../../00_Architecture/Trust_Model.md` |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-005) |
