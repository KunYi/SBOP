# SBOP System Overview

**Document ID:** ARC-SYS-001
**Version:** 2.0
**Status:** Approved
**Last Review:** 2026-04-28
**Owner:** System Architecture Team

---

## 1. Purpose

The SBOP (Secure Boot & OTA Platform) defines a hardware-agnostic security framework for ensuring firmware authenticity, integrity, and controlled deployment across embedded devices.

This document provides a high-level overview of system scope, objectives, and architectural boundaries.

---

## 2. System Objectives

The system is designed to achieve the following:

* Ensure only authenticated firmware is executed on devices
* Provide secure and reliable OTA update mechanisms
* Establish device identity and cryptographic trust anchors
* Prevent unauthorized cloning and rollback attacks
* Support scalable backend-driven device lifecycle management

---

## 3. System Scope

SBOP includes:

* Device-side secure boot and update logic
* Backend infrastructure for firmware distribution and key management
* Provisioning and manufacturing integration
* Security enforcement across firmware lifecycle

SBOP excludes:

* Application-level business logic
* Hardware-specific driver implementations
* Physical security mechanisms outside defined threat model

---

## 4. System Components

The system is composed of the following logical components:

* Boot Component (Secure Boot Execution)
* Update Component (OTA Management)
* Identity Component (Device Identity & Provisioning)
* Crypto Component (Key and Cryptographic Operations)
* Backend Component (Control Plane)
* Toolchain Component (Build, Signing, Deployment)

---

## 5. Trust Boundaries

The system defines the following trust boundaries:

* Device Boundary
* Backend Boundary
* Manufacturing Boundary

Each boundary enforces distinct security controls and trust assumptions.

---

## 6. Assumptions

* Devices have access to a unique hardware identifier or equivalent entropy source
* Cryptographic primitives are correctly implemented or provided
* Secure storage is available for root keys or key derivation

---

## 7. Non-Goals

* No guarantee against invasive physical attacks beyond defined threat model
* No dependency on specific MCU or hardware platform
* No runtime protection of application-layer vulnerabilities

---

## 8. Document References

* Requirements Specification
* System Design Specification
* Security Model
* Verification Plan

---

## 9. Next Stage — Planned Enhancements

The current SBOP specification (v2.0) matches or exceeds MCUboot across security depth and reliability. Three MCUboot capabilities are identified for the next specification revision. These are documented here as planned work, not as current requirements.

### 9.1 Multi-Signature Support

**Current state:** SBOP supports a single Ed25519 signature per firmware image. The SignatureBlock carries one public key, verified against one SR1 OTP fingerprint slot.

**Target:** Support for multiple independent signatures over the same image, enabling multi-party approval workflows (e.g., manufacturer + fleet operator both sign before deployment). This is analogous to MCUboot's TLV-chained signatures but without adopting the full TLV format.

**Design approach (v3.0 candidate):**

- Extend the image layout with a chained signature area after the SignatureBlock
- `flags` bit 3: `FLAG_MULTI_SIG` — when set, an additional MultiSigBlock follows the SignatureBlock
- Each additional signer provides: `{key_index, key_fingerprint, signature}` — verified against SR1 OTP in order
- All signatures must be valid for the image to pass Gate 2
- Backward-compatible: `FLAG_MULTI_SIG = 0` behaves identically to v2.0

**Impact:** SignatureBlock area grows by N × 68 bytes (1 B key_index + 3 B reserved + 64 B signature) per additional signer. Boot time increases by N × ~20 ms (Ed25519 verify). No change to key provisioning flow — additional signers use existing key slots 0-3.

### 9.2 PSA Crypto API & TF-M Integration

**Current state:** SBOP defines its own crypto API (`crypto_verify_signature`, `crypto_compute_hash`, `crypto_derive_key`, `crypto_constant_time_compare`) with opaque `KeyRef` handles. This is internally consistent but not aligned with the Arm PSA Crypto API.

**Target:** Map SBOP crypto operations to PSA Crypto API calls so SBOP can compile as a TF-M BL1 or run alongside a TF-M secure partition. This enables SBOP to leverage PSA-certified crypto implementations and integrate with TF-M's secure storage, attestation, and lifecycle management.

**Design approach (v3.0 candidate):**

- SBOP `KeyRef` → `psa_key_id_t` (already semantically compatible — both are opaque handles)
- `crypto_verify_signature` → `psa_sign_hash` with `PSA_ALG_SHA_256` + `PSA_ALG_ED25519_PURE`
- `crypto_compute_hash` → `psa_hash_compute` with `PSA_ALG_SHA_256`
- `crypto_derive_key` → `psa_key_derivation` (HKDF)
- `crypto_constant_time_compare` → PSA doesn't expose this; keep SBOP's implementation
- SBOP's hardware CRYP register zeroization maps to PSA's `psa_purge_key`
- KD derivation via SVC maps to TF-M secure partition IPC

**Impact:** Crypto subsystem implementation replaced with PSA API calls. The logical contract (constant-time, no key exposure, pre/post conditions) remains unchanged. SBOP's Interface_Contracts.md doesn't change — only the implementation layer. Estimated: ~1,500 bytes code reduction by delegating to PSA implementation.

**Compatibility matrix:**

| SBOP Concept | PSA Equivalent | Notes |
|---|---|---|
| KeyRef | psa_key_id_t | Both are opaque handles, usage-gated |
| KD (device key) | PSA key in secure storage | PSA provides persistent key storage |
| CRYP register key load | psa_aead_encrypt / psa_cipher_encrypt | PSA manages key in crypto engine |
| SVC-based key derivation | TF-M secure partition call | Same trust boundary via IPC |
| crypto_hw_key_zeroize() | psa_purge_key() | Both zeroize key material from hardware |

### 9.3 Multi-Image Support

**Current state:** SBOP manages a single firmware image per slot — one application binary. Secondary processors, FPGA bitstreams, or coprocessor firmware are out of scope.

**Target:** Manage N independent firmware images within the same slot structure, each with its own signature verification, version counter, and health tracking. This enables SBOP to be the single trust anchor for heterogeneous SoCs with multiple processing elements.

**Design approach (v3.0 candidate):**

- Image descriptor table at a fixed offset within the slot, listing N images
- Each image has: `{image_id, offset, size, hash, version}`
- Bootloader verifies all images before marking the slot valid
- `boot_set_confirmed()` confirms the entire slot (all images)
- Per-image versions tracked in the critical metadata journal
- Backward-compatible: N=1 behaves identically to v2.0

**Image descriptor table layout (candidate):**

```
Offset  Size   Field
0x00    2 B    image_count : u16          // Number of images in this slot
0x02    2 B    entry_size  : u16 = 64     // Size of each descriptor entry
0x04    N×64 B descriptors[]               // Sorted by image_id

Descriptor entry (64 bytes):
  0x00   4 B    image_id    : u32          // Unique image identifier
  0x04   4 B    offset      : u32          // Byte offset within slot
  0x08   4 B    size        : u32          // Image size in bytes
  0x0C   4 B    version     : u32          // Per-image monotonic version
  0x10  32 B    hash        : u8[32]       // SHA-256 of this image
  0x30   4 B    flags       : u32          // Per-image flags
  0x34  12 B    reserved    : u8[12]       // Must be zero
```

**Impact:** Slot size budget per image: N × 64 bytes for descriptor table. Boot time: N × (SHA-256 + Ed25519 verify per image). Critical journal grows by N × 52 bytes (additional ImageInfo entries). Backward-compatible with v2.0 single-image images (image_count=1, entry_size=0 → treated as legacy single-image format).

### 9.4 Priority & Sequencing

| Enhancement | Priority | Rationale | Dependencies |
|---|---|---|---|
| Multi-signature | High | Required for multi-party manufacturing + operator signing workflows | None |
| PSA Crypto / TF-M | Medium | Ecosystem alignment; required for TF-M-based platforms | None |
| Multi-image | Medium | Required for heterogeneous SoCs (app + DSP + FPGA) | May depend on PSA Crypto for per-image key management |

Recommended implementation order: multi-signature first (isolated change, high demand), then PSA Crypto alignment (enables TF-M integration), then multi-image (builds on both previous enhancements).
