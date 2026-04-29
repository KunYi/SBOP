# Reference Implementation Architecture

**Document ID:** PRD-REF-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28
**Owner:** Product Architecture Team

---

## 1. Purpose

This document defines the reference implementation architecture for SBOP. It bridges the Tier-1 specification to an implementable platform by specifying the concrete component breakdown, build pipeline, flash layout, and implementation guidance derived from the system specification.

This is the primary document for the **reference platform implementation phase**.

---

## 2. Architecture Overview

### 2.1 Component Map

```
┌──────────────────────────────────────────────────────────┐
│                    Host-Side Toolchain                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Image    │  │ Signing  │  │ SBOM     │  │ Provision │  │
│  │ Builder  │  │ Tool     │  │ Generator│  │ Station   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
└───────┼──────────────┼──────────────┼──────────────┼──────┘
        │              │              │              │
┌───────▼──────────────▼──────────────▼──────────────▼──────┐
│                    Backend Server                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Firmware │  │ Device   │  │ Key      │  │ Deploy   │  │
│  │ Registry │  │ Registry │  │ Manager  │  │ Controller│  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└───────────────────────┬──────────────────────────────────┘
                        │ TLS 1.3 + Mutual Auth
┌───────────────────────▼──────────────────────────────────┐
│                   Device Firmware                          │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Zone 1: Secure Core (Immutable / Protected)          │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │ │
│  │  │ Boot   │ │ Crypto │ │Identity│ │ Secure │        │ │
│  │  │ Manager│ │ Engine │ │ Manager│ │ Debug  │        │ │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘        │ │
│  └──────┼──────────┼──────────┼──────────┼─────────────┘ │
│         │          │          │          │                │
│  ┌──────▼──────────▼──────────▼──────────▼─────────────┐ │
│  │ Zone 2: Application                                   │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │ │
│  │  │ OTA    │ │ App    │ │ Tele-  │ │ Storage│        │ │
│  │  │ Client │ │ Logic  │ │ metry  │ │ Driver │        │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘        │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ Hardware Abstraction Layer (HAL)                      │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │ │
│  │  │ Flash  │ │ Crypto │ │ TRNG   │ │ Watch- │        │ │
│  │  │ Driver │ │ Accel  │ │ Driver │ │ dog    │        │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘        │ │
│  └──────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

### 2.2 Specification References

| Component | Primary Spec | Interface Contract | Data Structures |
| --- | --- | --- | --- |
| Boot Manager | `Boot_Flow_Pseudocode.md` | `Interface_Contracts.md` §3 | `Data_Structures.md` §3-6 |
| Crypto Engine | `Crypto_Algorithms.md` | `Interface_Contracts.md` §4 | KeyRef, KeyMaterial |
| Identity Manager | `Provisioning_Flow.md` | `Interface_Contracts.md` §6 | DeviceID, ProvisioningData |
| OTA Client | `OTA_Flow_Pseudocode.md` | `Interface_Contracts.md` §5 | UpdateInfo, AuthToken |
| Secure Debug | `Secure_Debug_Architecture.md` | `Interface_Contracts.md` | DebugAuthChallenge |
| Error Handling | `Error_Code_Catalog.md` | — | All error types |
| Image Format | `Image_Format.md` | — | ImageHeader, SignatureBlock |

---

## 3. Flash Layout

### 3.1 Physical Layout

```
┌──────────────────────────────────────┐  0x0000_0000
│ Boot ROM / Immutable Boot            │  (MCU hardware)
├──────────────────────────────────────┤
│ SBOP Bootloader (Zone 1)             │  64 KB
│  - Boot manager                      │
│  - Crypto engine                     │
│  - Identity manager                  │
├──────────────────────────────────────┤
│ Bootloader Config + Keys             │  4 KB
│  - Key storage (secure area)         │
│  - Version counter (OTP)             │
│  - Boot slot selection               │
│  - Tamper log                        │
├──────────────────────────────────────┤
│ Slot Metadata A                      │  1 KB
├──────────────────────────────────────┤
│ Firmware Slot A                      │  (image_size)
│  - ImageHeader                       │
│  - Payload                           │
│  - SignatureBlock                    │
├──────────────────────────────────────┤
│ Slot Metadata B                      │  1 KB
├──────────────────────────────────────┤
│ Firmware Slot B                      │  (image_size)
├──────────────────────────────────────┤
│ Application Data / Filesystem        │  remainder
└──────────────────────────────────────┘
```

### 3.2 Slot Metadata

```
struct SlotMetadata {
    magic:            u32 = 0x534C4F54 ("SLOT")
    slot_status:      u8,    // SlotStatus enum
    firmware_version: u32,
    image_hash:       [u8; 32],
    boot_count:       u32,
    last_boot_error:  u32,
    crc32:            u32,
}
```

### 3.3 Memory Protection

| Region | Permissions | Zone Access |
| --- | --- | --- |
| Bootloader code | RX | Zone 1 only |
| Bootloader data | RW | Zone 1 only |
| Key storage | R (via secure API) | Zone 1 only |
| Slot A | RX | Both (read via HAL) |
| Slot B | RX | Both (read via HAL) |
| Slot metadata | RW (via HAL) | Zone 1 write; Zone 2 read |
| Application RAM | RWX | Zone 2 |

---

## 4. Build Pipeline

### 4.1 Build Steps

```
Step 1: Compile bootloader (Zone 1)
        → Output: sbop_bootloader.bin

Step 2: Compile application firmware (Zone 2)
        → Output: application.bin

Step 3: Generate SBOM
        → Output: sbom.spdx.json

Step 4: Package firmware image
        tool: sbop-pack
        Input: application.bin, version, flags
        Output: firmware.sbop (ImageHeader + payload)

Step 5: Sign firmware image
        tool: sbop-sign
        Input: firmware.sbop, signing_key (from HSM)
        Output: firmware_signed.sbop (ImageHeader + payload + SignatureBlock)

Step 6: Verify signed image
        tool: sbop-verify
        Input: firmware_signed.sbop, verify_key
        Check: signature, hash, header validity

Step 7: Register with backend
        tool: sbop-register
        Input: firmware_signed.sbop, metadata, sbom.spdx.json
        Upload to firmware registry
```

### 4.2 Host Tools

| Tool | Purpose | Specification Reference |
| --- | --- | --- |
| `sbop-pack` | Construct image header + payload | `Image_Format.md` |
| `sbop-sign` | Sign image with HSM-backed key | `Supply_Chain_Security.md` §4 |
| `sbop-verify` | Verify image signature and hash | `Crypto_Algorithms.md` |
| `sbop-sbom` | Generate SPDX/CycloneDX SBOM | `Supply_Chain_Security.md` §6 |
| `sbop-provision` | Manufacturing provisioning tool | `Manufacturing_Security.md` §4 |

---

## 5. Boot Flow (Reference Implementation)

→ See `03_Subsystem/Boot/Boot_Flow_Pseudocode.md` for the complete algorithm.

### 5.1 Implementation Notes

| Pseudocode Phase | Implementation Guidance |
| --- | --- |
| INIT_HARDWARE | Initialize watchdog, clock, flash controller. Sample tamper sensors. |
| SELECT_SLOT | Read boot slot selection from metadata region (atomic). |
| LOAD_IMAGE | DMA from flash to RAM. Validate minimum size. |
| PARSE_HEADER | Parse `ImageHeader` struct. Validate all fields per `Data_Structures.md` §3.2. |
| VERIFY_SIGNATURE | Use hardware crypto accelerator if available. Ed25519. Must be constant-time per `Side_Channel_Countermeasures.md`. |
| VERIFY_INTEGRITY | SHA-256 via hardware accelerator. Compare with `constant_time_compare`. |
| CHECK_VERSION | Read version counter from OTP/secure flash. Redundant read for fault detection. |
| COMMIT_VERSION | Write new version to OTP if higher. |
| MARK_ACTIVE | Atomic write of slot status. Must survive power loss mid-write. |
| LOCK_BOOT | Enable MPU/MMU to protect Zone 1 memory. Disable boot-mode peripherals. |
| EXECUTE | Set MSP to application stack, set PC to entry point. Clear registers first. |

### 5.2 Error Flow

All `assert()` failures in pseudocode → `handle_boot_error()` → FAILSAFE.
→ See `Error_Code_Catalog.md` §4 for all boot error codes.
→ See `Boot_Failure_Model.md` for failure scenarios.

---

## 6. OTA Flow (Reference Implementation)

→ See `03_Subsystem/Update/OTA_Flow_Pseudocode.md` for the complete algorithm.

### 6.1 Transport Layer

| Requirement | Recommendation |
| --- | --- |
| TLS library | mbedTLS 3.x or wolfSSL 5.x |
| Mutual auth | Client certificate (device KD-derived) |
| Cipher suite | TLS 1.3: TLS_AES_128_GCM_SHA256 |
| Download resume | HTTP Range requests for interrupted downloads |

### 6.2 Rollback Integration

OTA rollback is triggered by boot failure detection:
1. Boot attempts to verify+execute new image
2. If boot fails → enters FAILSAFE
3. On next power cycle, boot detects PENDING slot was never marked ACTIVE
4. Boot switches to FALLBACK slot
5. If fallback succeeds, reports rollback to backend via telemetry

---

## 7. Crypto Implementation Guidance

| Algorithm | Implementation | Notes |
| --- | --- | --- |
| Ed25519 | libsodium, TweetNaCl, or monocypher | Must verify in constant time per RFC 8032 §5.1 |
| SHA-256 | mbedTLS `sha256.h` or wolfSSL `sha256.h` | Hardware acceleration recommended |
| HKDF | mbedTLS `hkdf.h` or wolfSSL `hkdf.h` | Salt must be random + device-specific |
| TRNG | MCU hardware TRNG with health tests | NIST SP 800-90B compliant |

---

## 8. Target Platform Characteristics

### 8.1 Minimum Requirements

| Resource | Minimum | Recommended |
| --- | --- | --- |
| Flash (bootloader) | 64 KB | 128 KB |
| Flash (per slot) | Application-dependent | — |
| RAM | 32 KB | 64 KB |
| CPU | Cortex-M0+ / RISC-V RV32IMC | Cortex-M33 with TrustZone |
| Secure storage | 4 KB OTP or write-protected flash | Secure element with monotonic counters |
| TRNG | Required | NIST SP 800-90B compliant |
| Crypto accelerator | Optional | AES + SHA-256 hardware |
| MPU | Optional | Required for Zone 1/2 isolation |

### 8.2 Recommended Reference Platforms

| Platform | Architecture | Security Features |
| --- | --- | --- |
| STM32L5 / STM32U5 | Cortex-M33 + TrustZone | HW crypto, secure storage, active tamper |
| NXP LPC55S69 | Cortex-M33 + TrustZone | PUF, PRINCE flash encryption, CASPER crypto |
| Microchip SAM L11 | Cortex-M23 + TrustZone | ARMv8-M TrustZone, secure boot ROM |
| ESP32-C3 / C6 | RISC-V RV32IMC | HW crypto, secure boot, flash encryption |
| OpenTitan Earl Grey | RISC-V RV32IMC | Open-source silicon root of trust |

---

## 9. Development Phases

| Phase | Deliverable | Status |
| --- | --- | --- |
| Phase 0: Specification | Complete Tier-1 spec (85 documents) | **Done** |
| Phase 1: Core Bootloader | `sbop_boot` reference implementation on target platform | Next |
| Phase 2: Crypto Integration | Constant-time crypto, KDF, signature verification | Next |
| Phase 3: OTA Client | Download, verify, install with rollback | Phase 2 + |
| Phase 4: Host Tools | `sbop-pack`, `sbop-sign`, `sbop-sbom` | Phase 1 + |
| Phase 5: Backend | Firmware registry, device registry, deployment controller | Phase 4 + |
| Phase 6: Provisioning | Manufacturing station integration | Phase 1 + |
| Phase 7: Certification Evidence | Test execution, coverage reports, timing measurements | Phase 3 + |

---

## 10. Security Controls Checklist (Pre-Implementation)

Before implementation begins, confirm that the specification provides:

- [x] Boot flow with exact state machine transitions
- [x] Image format with byte-level layout
- [x] Error codes for all failure modes
- [x] Interface contracts with pre/post conditions
- [x] Data structure definitions with sizes
- [x] Cryptographic algorithm selection with rationale
- [x] Key hierarchy (KR → KD → KI)
- [x] Debug port lifecycle definition
- [x] Tamper detection and response specification
- [x] Side-channel countermeasure requirements
- [x] Supply chain security requirements
- [x] Manufacturing security and provisioning protocol
- [x] Standards compliance mapping (6 standards)
- [x] Risk assessment (TARA + risk quantification)
- [x] Attack tree with 11 root branches
