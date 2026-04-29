# Architecture Decision Record

**Document ID:** ARC-ADR-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28
**Owner:** System Architecture Team

---

## 1. Purpose

This document records the key architectural decisions made during SBOP specification. Each decision includes context, rationale, alternatives considered, and consequences. This is a living document — new decisions are appended as the architecture evolves.

---

## 2. Decision Records

### ADR-001: Dual-Image A/B Slot Model

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, Product Architecture

**Context:**
SBOP must support firmware updates that are recoverable under all failure conditions, including power loss mid-update, corrupted images, and authentication failures. The update mechanism must work over constrained links (space telemetry, industrial field buses).

**Decision:**
Use a dual-image A/B slot model where:
- One slot is always ACTIVE (currently executing firmware)
- The other slot receives updates (INACTIVE → PENDING → ACTIVE)
- The bootloader selects the ACTIVE slot and falls back to the previous slot on failure
- Slot state transitions are atomic

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Single-image with recovery partition | Recovery partition wastes flash; no atomic switch |
| Golden image in ROM | Cannot update; defeats OTA purpose |
| External recovery via debug | Requires physical access; not feasible for deployed devices |
| Three-slot model | Higher flash overhead; marginal benefit over A/B |

**Consequences:**
- Doubles flash requirement for firmware storage
- Requires slot metadata with CRC for integrity
- Enables atomic rollback without recovery partition
- Simplifies OTA recovery logic

**References:** `Boot_Flow_Pseudocode.md`, `OTA_Flow_Pseudocode.md`, `Unified_State_Model.md`

---

### ADR-002: Single Signature Algorithm (Ed25519)

**Status:** Accepted
**Date:** 2026-04-28
**Revised:** 2026-04-29 — simplified to Ed25519-only
**Stakeholders:** Security Architecture, Crypto Team

**Context:**
Firmware signature verification is the single most critical security operation. Algorithm choice affects security level, code size, verification speed, and standards compliance.

**Decision:**
Use Ed25519 as the single required firmware signature algorithm. ECDSA P-256 is removed to eliminate redundant algorithms with similar security properties. Ed25519 is FIPS 186-5 compliant, simpler to implement correctly, and more resistant to side-channel attacks than ECDSA.

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| ECDSA P-256 only | More complex (ASN.1 DER, variable-length signatures), nonce-reuse risk, slower verification |
| Dual algorithm (Ed25519 + ECDSA P-256) | Redundant — same 128-bit security level; doubled code size, attack surface, and certification burden |
| RSA-2048 | Signature size too large for embedded storage; slower verification |
| Custom crypto | Prohibited by security policy |

**Consequences:**
- Single algorithm to implement, test, and certify
- Smaller code size (~4 KB vs ~12 KB for dual)
- FIPS 186-5 compliance via Ed25519 (§7, Edwards-curve digital signature)
- SignatureBlock.algorithm must be 0x02 (Ed25519)

**References:** `Crypto_Algorithms.md`, `Signature_Format.md`, `Data_Structures.md` §4

---

### ADR-003: Constant-Time Requirement on All Security-Critical Paths

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, Crypto Team

**Context:**
Secure boot verification is vulnerable to timing side-channels. Any data-dependent timing variation in signature verification, hash comparison, or version checking can leak information about the expected values, enabling attackers to forge valid signatures or bypass verification.

**Decision:**
All security-critical operations must be constant-time:
- `crypto_verify_signature()` — no early return, fixed-sequence operations
- `crypto_compare_hash()` — byte-level constant-time comparison
- Version check — fixed-sequence, no short-circuit evaluation
- Image header parsing — fixed-path, no data-dependent branches

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Best-effort (mitigate only obvious leaks) | Certification requires quantifiable assurance |
| Hardware-only countermeasures | Not available on all target platforms |
| Accept timing risk at SL 1-2 | Even low-SL devices may process safety-critical functions |

**Consequences:**
- Performance overhead: ~5-10% slower verification (acceptable for boot-time operation)
- Requires per-platform timing measurement (TIM-001, TIM-002)
- Precludes use of non-constant-time third-party crypto libraries
- Enables TVLA (Test Vector Leakage Assessment) pass at SL 3+

**References:** `Side_Channel_Countermeasures.md`, `Interface_Contracts.md` §crypto_verify_signature, `Crypto_Algorithms.md`

---

### ADR-004: Zone 1 / Zone 2 Separation with MPU/MMU Enforcement

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, System Architecture

**Context:**
The bootloader (Zone 1) handles cryptographic keys, signature verification, and slot selection. The application (Zone 2) handles OTA, telemetry, and application logic. A compromise in Zone 2 must not allow an attacker to bypass boot verification or extract keys. The separation must be enforceable on Cortex-M class hardware with an MPU.

**Decision:**
- Zone 1: Bootloader, crypto engine, identity manager, secure debug — locked read-only after boot
- Zone 2: Application firmware, OTA client — no access to Zone 1 memory
- MPU/MMU configured as final step of boot before handoff
- Zone 2 may read slot metadata via HAL (read-only) but never write active slot selection

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Single flat address space | No isolation; any Zone 2 bug compromises entire device |
| TrustZone-M (secure/non-secure worlds) | Not available on Cortex-M0+, RISC-V without PMP |
| Software sandboxing (e.g., WebAssembly) | Performance overhead too high for resource-constrained MCUs |

**Consequences:**
- Requires MPU on target platform (8 regions minimum)
- Zone 1 must not expose any writable interface to Zone 2
- Zone 2 cannot directly access flash controller for slot management
- Aligns with IEC 62304 segregation requirements and IEC 62443 zone model

**References:** `Reference_Architecture.md` §3.3, `Zone_Conduit_Model.md`, `Boot_Flow_Pseudocode.md` Phase 9 (LOCK_BOOT)

---

### ADR-005: Monotonic Version Counter in OTP / Write-Protected Storage

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture

**Context:**
Anti-rollback protection prevents an attacker from reinstalling an older, vulnerable firmware version that was validly signed. The version counter must survive power cycles, cannot be decremented by software, and must be resistant to fault injection.

**Decision:**
- Version counter stored in OTP (one-time programmable) memory or write-protected secure flash
- Monotonic rule: new firmware version MUST be > current counter value (not ≥)
- Redundant read with fault detection (two reads, compare, fault if mismatch)
- Counter programmed after successful verification (COMMIT_VERSION phase)

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Rewritable flash with software-only protection | Vulnerable to flash manipulation via debug/fault injection |
| Hardware monotonic counter in secure element | Preferred if available, but not required (may not exist on all platforms) |
| Version in signed image only (no device-side counter) | Attacker can replay old signed image; no device-side enforcement |
| Backend-enforced only | Device offline scenarios; backend may not see all boots |

**Consequences:**
- Requires OTP or write-protected flash region (minimum 4 bytes per version field)
- Once a version is burned, that OTP cell is consumed (cannot be reused)
- Version counter exhaustion possible over very long device lifetimes (mitigated by 32-bit counter: ~4 billion updates)
- Boot must handle the case where OTP write fails mid-operation

**References:** `Boot_Flow_Pseudocode.md` Phase 6-7, `Data_Structures.md` §4.2, `Unified_State_Model.md`

---

### ADR-006: HSM-Backed Firmware Signing with Quorum Authorization

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, Operations

**Context:**
The firmware signing key is the highest-value asset in the SBOP ecosystem. Compromise allows an attacker to sign arbitrary firmware for all devices. The signing infrastructure must prevent both external attacks and insider threats.

**Decision:**
- Signing key generated and stored exclusively in FIPS 140-2 Level 3+ HSM
- Key never leaves HSM — signing operations performed inside HSM boundary
- Multi-party quorum required: N-of-M operators must authorize each signing operation
- All signing operations logged with operator identity and timestamp
- Signing ceremony procedure documented and rehearsed
- Air-gapped signing environment for production keys

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Software-only signing (e.g., openssl) | Key exposed to build server compromise |
| HSM without quorum | Insider with HSM access can sign alone |
| Cloud KMS (e.g., AWS KMS) | Shared responsibility model — not acceptable for safety-critical devices |
| Per-device signing keys | Backend complexity; does not scale |

**Consequences:**
- Signing ceremony adds operational overhead (~30 min per release)
- Requires at least 3 operators for quorum (2-of-3 minimum)
- HSM procurement and maintenance cost
- Signing cannot be fully automated in CI/CD (by design)

**References:** `Supply_Chain_Security.md` §4, `Key_Management.md`

---

### ADR-007: Challenge-Response Debug Authentication with Device-Specific Key

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, Manufacturing

**Context:**
Debug ports (JTAG/SWD) provide full access to device memory and execution. They must be disabled in production but may need to be re-enabled for failure analysis. A debug unlock mechanism must not allow an attacker to extract keys or bypass secure boot.

**Decision:**
- Debug port permanently disabled (fuse blown) as final provisioning step
- If debug re-enablement is required: challenge-response protocol using device-specific KD_Debug key
- KD_Debug derived from root key (KR) + device UID via HKDF — unique per device
- Rate-limited authentication (exponential backoff) with permanent lockout after 10 failures
- Authorized unlock tokens issued by backend, time-limited, device-specific
- Key material always protected from debug access, regardless of debug unlock state

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Fixed debug password | One compromise affects all devices |
| Debug permanently disabled (no unlock) | Prevents failure analysis of field-returned devices |
| Debug always available (trusted environment only) | Manufacturing station compromise exposes all devices |
| JTAG boundary scan only (no CPU debug) | Insufficient for complex failure analysis |

**Consequences:**
- Requires non-volatile storage for failure counter and lockout state
- Permanent lockout is irreversible — locked devices cannot be debugged
- Backend must maintain KD_Debug derivation capability for all devices
- Manufacturing stations must not store KD_Debug after provisioning

**References:** `Secure_Debug_Architecture.md`, `Manufacturing_Security.md` §6

---

### ADR-008: Deterministic, Reproducible Builds

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, Build Infrastructure

**Context:**
Build server compromise is a supply chain attack vector. If an attacker can inject code during the build process, the resulting firmware will pass signature verification. Reproducible builds allow independent verification that the signed binary matches the source code.

**Decision:**
- Build must be bit-for-bit reproducible: identical source + identical toolchain → identical binary
- All build inputs pinned: compiler version, libraries, build flags, environment
- SBOM generated at build time (SPDX + CycloneDX)
- Build output includes verifiable hash that can be independently reproduced
- Ephemeral build environment (container/VM), destroyed after each build

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Trusted build server only | Insider threat; single point of compromise |
| Post-build binary diffing | Cannot prove binary derived from claimed source |
| Build attestation without reproducibility | Attestation can be forged if build server is compromised |

**Consequences:**
- Requires deterministic toolchain (compiler, linker, packer)
- Build metadata must be archived alongside firmware
- Timestamps and non-deterministic identifiers must be stripped or normalized
- Increases build complexity (environment pinning)

**References:** `Supply_Chain_Security.md` §3.1, `Reference_Architecture.md` §4

---

### ADR-009: Pseudocode-Driven Specification (Not Reference Code)

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Product Architecture, Implementation Teams

**Context:**
SBOP is a hardware-agnostic specification. Providing a reference implementation in a specific language (C, Rust) would bias implementers toward that language and platform. However, the specification must be precise enough to be implementable without ambiguity.

**Decision:**
- Critical algorithms specified in structured pseudocode with explicit state transitions
- Pseudocode uses guard conditions, explicit error paths, and timing annotations
- Interface contracts use IDL-style format with pre/post conditions
- Data structures defined with exact sizes and serialization rules
- Reference implementation on specific platforms is a separate deliverable (Phase 1+)

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Reference C implementation as spec | Biases toward C; harder to verify spec independently |
| Formal specification language (Z, VDM) | Barrier to entry for embedded teams; tool ecosystem limited |
| Natural language only | Ambiguous; leads to implementation divergence |

**Consequences:**
- Pseudocode must be manually translated to target language
- Risk of translation errors (mitigated by test vectors and formal contracts)
- Reference implementation serves as validation, not specification
- Enables independent implementations for different platforms

**References:** `Boot_Flow_Pseudocode.md`, `OTA_Flow_Pseudocode.md`, `Interface_Contracts.md`

---

### ADR-010: Device Identity Anchored to Hardware Entropy (TRNG + Optional PUF)

**Status:** Accepted
**Date:** 2026-04-28
**Stakeholders:** Security Architecture, Manufacturing

**Context:**
Device cloning undermines anti-counterfeiting and enables unauthorized devices on the network. Each device must have a unique, non-transferable identity. If identity can be copied from one device to another, the security model collapses.

**Decision:**
- Unique Device ID (UID) generated from hardware TRNG during provisioning (128-bit minimum entropy)
- PUF (Physically Unclonable Function) preferred if available on target platform — provides implicit key storage
- Device Key (KD) derived from KR + UID via HKDF — binds key material to device identity
- Backend validates UID uniqueness at registration time
- Identity locked irreversibly after provisioning — no modification possible

**Alternatives considered:**

| Alternative | Rejected Because |
| --- | --- |
| Pre-programmed serial number | Cloneable; no entropy guarantee |
| MAC address as identity | Cloneable; not cryptographically unique |
| Certificate-based without hardware binding | Certificate can be copied to cloned device |
| PUF-only (no UID) | PUF reliability varies; needs error correction |

**Consequences:**
- Requires working TRNG at provisioning time
- PUF enrollment adds provisioning step if PUF is used
- Backend must handle duplicate UID detection (extremely unlikely with 128-bit TRNG)
- Identity cannot be changed after lock — device is permanently bound to its identity

**References:** `Identity_Overview.md`, `Provisioning_Flow.md`, `Manufacturing_Security.md` §4

---

## 3. Decision Status Legend

| Status | Meaning |
| --- | --- |
| Proposed | Under discussion, not yet decided |
| Accepted | Approved and implemented in specification |
| Deprecated | Was accepted, now superseded (see Superseded By) |
| Superseded | Replaced by a newer ADR |

---

## 4. Traceability

| ADR | Requirements Affected | Design Documents |
| --- | --- | --- |
| ADR-001 | REQ-FR-OTA-004, REQ-FR-OTA-005 | Boot_Flow_Pseudocode, OTA_Flow_Pseudocode, Unified_State_Model |
| ADR-002 | REQ-SR-BOOT-001 | Crypto_Algorithms, Image_Format, Data_Structures |
| ADR-003 | REQ-SR-SCH-001 | Side_Channel_Countermeasures, Interface_Contracts |
| ADR-004 | REQ-SR-BOOT-001, REQ-SR-KEY-001 | Reference_Architecture, Zone_Conduit_Model |
| ADR-005 | REQ-FR-BOOT-004, REQ-SR-OTA-002 | Boot_Flow_Pseudocode, Unified_State_Model |
| ADR-006 | REQ-SR-KEY-001, REQ-SR-SC-001 | Supply_Chain_Security, Key_Management |
| ADR-007 | REQ-SR-DBG-001, REQ-SR-DBG-002 | Secure_Debug_Architecture, Manufacturing_Security |
| ADR-008 | REQ-SR-SC-001 | Supply_Chain_Security, Reference_Architecture |
| ADR-009 | All | Boot_Flow_Pseudocode, OTA_Flow_Pseudocode, Interface_Contracts |
| ADR-010 | REQ-SR-ID-001, REQ-SR-ID-002 | Identity_Overview, Provisioning_Flow |
