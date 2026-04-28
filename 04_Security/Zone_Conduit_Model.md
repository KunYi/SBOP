# SBOP Zone and Conduit Model (IEC 62443)

**Document ID:** SEC-ZONE-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the security zone and conduit model for the SBOP platform in accordance with IEC 62443-1-1. It identifies zones of trust, communication conduits between zones, and the security requirements applicable to each.

---

## 2. Zone Definitions

### 2.1 Zone Overview

```
┌──────────────────────────────────────────────────────┐
│ Zone 6: Toolchain / Build                            │
│  ┌──────────────────────┐                            │
│  │ Build, Sign, Package │                            │
│  └──────────┬───────────┘                            │
└─────────────┼────────────────────────────────────────┘
              │ Conduit B: Image Distribution
┌─────────────▼────────────────────────────────────────┐
│ Zone 3: Backend Infrastructure                       │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐     │
│  │ Firmware │  │ Device   │  │ Key Management │     │
│  │ Registry │  │ Registry │  │ Server         │     │
│  └────┬─────┘  └────┬─────┘  └───────┬────────┘     │
└───────┼──────────────┼───────────────┼──────────────┘
        │ Conduit A: OTA Communication
┌───────▼──────────────▼───────────────▼──────────────┐
│ Zone 5: Network / Communication                      │
│  Untrusted transport (TLS tunnel)                    │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│ Zone 1: Device Core                                  │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐     │
│  │ Boot     │  │ Crypto   │  │ Identity       │     │
│  │ Manager  │  │ Engine   │  │ Manager        │     │
│  └────┬─────┘  └────┬─────┘  └───────┬────────┘     │
└───────┼──────────────┼───────────────┼──────────────┘
        │ Conduit D: Internal IPC
┌───────▼──────────────▼───────────────▼──────────────┐
│ Zone 2: Device Application                           │
│  ┌──────────────────────────────────────────┐        │
│  │ Application Firmware / Runtime            │        │
│  └──────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ Zone 4: Manufacturing / Provisioning                  │
│  ┌──────────────────────────────────────────┐        │
│  │ Provisioning Station, HSM, Key Injection  │        │
│  └──────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────┘
        Conduit C: Provisioning Channel (to Zone 1)
```

### 2.2 Zone 1: Device Core

**Description:** Contains the secure boot, cryptographic engine, and device identity components. This is the most trusted zone on the device.

**Assets:** Bootloader, crypto engine, identity manager, root key (KR), device key (KD), version counter, boot state machine.

**Security Characteristics:**
- Root of trust resides here
- All firmware verification occurs here
- Key material is protected (never exposed outside zone)
- State transitions are strictly enforced

**Trust Level:** Highest (implicitly trusted by all other zones)

### 2.3 Zone 2: Device Application

**Description:** Contains the application firmware and runtime environment. Trusted only after verification by Zone 1.

**Assets:** Application firmware, runtime data, OTA manager (update orchestration logic).

**Security Characteristics:**
- Receives verified firmware handoff from Zone 1
- May initiate OTA update requests (but verification delegated to Zone 1)
- Runtime isolation from Zone 1

**Trust Level:** Moderate (trusted after boot verification)

### 2.4 Zone 3: Backend Infrastructure

**Description:** Cloud/server-side infrastructure for firmware distribution, device management, and key lifecycle.

**Assets:** Firmware registry, device registry, signing keys, key management server, deployment orchestrator.

**Security Characteristics:**
- Authenticates devices before serving updates
- Enforces deployment policies (canary, rollback thresholds)
- Manages signing key lifecycle

**Trust Level:** Trusted but controlled (must authenticate devices; must not expose keys)

### 2.5 Zone 4: Manufacturing / Provisioning

**Description:** Physical factory environment where devices are provisioned with identity and keys.

**Assets:** Provisioning station, Hardware Security Module (HSM), device-unique identity data, root key material.

**Security Characteristics:**
- Physically secure environment
- Operator authentication and audit
- Key injection into device secure storage
- Permanent lock after provisioning

**Trust Level:** Trusted during provisioning only; untrusted after device leaves factory

### 2.6 Zone 5: Network / Communication

**Description:** The communication channel(s) between device and backend. Includes all intermediate network infrastructure.

**Assets:** OTA firmware payloads (in transit), authentication tokens, telemetry data.

**Security Characteristics:**
- Untrusted by default
- All traffic must be authenticated and integrity-protected
- Optional confidentiality for sensitive payloads
- Device must not trust network path

**Trust Level:** Untrusted (zero-trust communication)

### 2.7 Zone 6: Toolchain / Build

**Description:** The software development and build environment where firmware is built, tested, and signed.

**Assets:** Source code, build scripts, signing keys (at signing time), build artifacts.

**Security Characteristics:**
- Build must be reproducible
- Signing must use offline or HSM-protected keys
- SBOM must be generated at build time
- Build artifacts must be integrity-verified

**Trust Level:** Trusted but must enforce integrity and audit

---

## 3. Conduit Definitions

### 3.1 Conduit A: Device ↔ Backend (OTA Communication)

| Property | Requirement |
| --- | --- |
| Endpoints | Zone 1 (Device Core) ↔ Zone 3 (Backend) |
| Physical Path | Via Zone 5 (Network) |
| Authentication | Device identity (UID + KD-based challenge-response) |
| Integrity | TLS 1.3 or equivalent, plus application-layer firmware signature |
| Confidentiality | Optional (encrypted firmware if enabled via image flags) |
| Non-repudiation | Firmware signing provides non-repudiation of image origin |
| Availability | Redundant backend, retry with exponential backoff |
| SL-T | SL 2 |

### 3.2 Conduit B: Backend ↔ Toolchain (Image Distribution)

| Property | Requirement |
| --- | --- |
| Endpoints | Zone 6 (Toolchain) → Zone 3 (Backend) |
| Authentication | Mutual (build system identity + operator auth) |
| Integrity | Signed image integrity verified by backend before acceptance |
| Confidentiality | Encrypted channel for signed firmware upload |
| Non-repudiation | Build provenance record with signing metadata |
| SL-T | SL 3 |

### 3.3 Conduit C: Manufacturing ↔ Device (Provisioning)

| Property | Requirement |
| --- | --- |
| Endpoints | Zone 4 (Manufacturing) → Zone 1 (Device Core) |
| Physical Path | Physical connection (JTAG/SWD or equivalent) during manufacturing |
| Authentication | Mutual authentication between provisioning station and device |
| Integrity | Encrypted provisioning channel |
| Confidentiality | Key material injected over encrypted link only |
| Post-provisioning | Conduit permanently disabled |
| SL-T | SL 3 |

### 3.4 Conduit D: Device Core ↔ Device Application (Internal IPC)

| Property | Requirement |
| --- | --- |
| Endpoints | Zone 1 (Boot/Crypto/Identity) → Zone 2 (Application) |
| Physical Path | On-device memory bus / internal IPC |
| Direction | Zone 1 → Zone 2 (handoff only; Zone 2 cannot call back into Zone 1) |
| Authentication | Zone 1 verifies application before handoff |
| Integrity | Memory protection between zones |
| SL-T | SL 2 |

---

## 4. Zone Security Requirements

### 4.1 Per-Zone Security Control Allocation

| Zone | Authentication | Integrity | Confidentiality | Monitoring | Recovery |
| --- | --- | --- | --- | --- | --- |
| Zone 1 | Signature verification, device identity | Hash verification, state machine | Key isolation | Status signals | FAILSAFE |
| Zone 2 | Verifies as needed | App-level (out of scope) | App-level (out of scope) | Telemetry | OTA revert |
| Zone 3 | Device auth, operator auth | Storage integrity | Encrypted at rest | Security monitoring | Redundancy |
| Zone 4 | Operator auth, device auth during provisioning | Audit logging | Encrypted key injection | Audit trail | Re-provision |
| Zone 5 | None (untrusted) | TLS, app-level signature | TLS encryption | Network monitoring | — |
| Zone 6 | Operator auth, build system auth | Reproducible builds, signed artifacts | Source protection | Build audit | — |

---

## 5. Inter-Zone Trust Relationships

### 5.1 Trust Matrix

| From \ To | Zone 1 | Zone 2 | Zone 3 | Zone 4 | Zone 5 | Zone 6 |
| --- | --- | --- | --- | --- | --- | --- |
| **Zone 1** | — | Trusted (verified) | Authenticated | Trusted (during provisioning) | — | — |
| **Zone 2** | Not trusted (cannot call) | — | Authenticated (via Zone 1) | — | — | — |
| **Zone 3** | Authenticated | — | — | Not accessible | Not trusted | Authenticated |
| **Zone 4** | Physically connected | — | Authenticated | — | — | — |
| **Zone 5** | Not trusted | Not trusted | Not trusted | Not trusted | — | Not trusted |
| **Zone 6** | Not accessible | Not accessible | Authenticated | Not accessible | Not trusted | — |

### 5.2 Communication Rules

1. **Zone 1 never communicates outward** except via status signals (fail-safe indicator)
2. **Zone 2 may initiate Conduit A** (OTA request) but **Zone 1 performs verification**
3. **Zone 3 never initiates Conduit C** (manufacturing is one-directional: Zone 4 → Zone 1)
4. **Conduit C is physically severed after provisioning** (debug port disabled, fuses blown)
5. **All Zone 5 traffic is treated as hostile** — zero-trust communication enforced end-to-end

---

## 6. Zone-to-Security-Level Mapping

| Zone ID | Zone Name | SL-T | Rationale |
| --- | --- | --- | --- |
| Zone 1 | Device Core | **SL 3** | Root of trust; compromise grants persistent access. Requires sophisticated attacker resistance. |
| Zone 2 | Device Application | **SL 2** | Application-level; compromise is contained by Zone 1 boundaries. |
| Zone 3 | Backend Infrastructure | **SL 3** | Controls fleet-wide firmware distribution and key management. High-value target. |
| Zone 4 | Manufacturing / Provisioning | **SL 2** | Physically secure + time-limited. Controls initial trust establishment. |
| Zone 5 | Network / Communication | **SL 2** | Untrusted by design; end-to-end protection at application layer. |
| Zone 6 | Toolchain / Build | **SL 3** | Compromise enables supply chain attack at scale. Requires sophisticated protection. |

---

## 7. Zone Boundary Enforcement

### 7.1 Zone 1 → Zone 2 Boundary

- Boot verifies application before transferring control
- Memory protection prevents Zone 2 from accessing Zone 1 code/data
- Zone 1 memory is read-only after boot completes

### 7.2 Zone 3 → Zone 5 Boundary

- Backend enforces TLS with client certificate validation
- Rate limiting on device authentication
- Anomaly detection on update request patterns

### 7.3 Zone 4 → Zone 1 Boundary (Provisioning)

- One-time provisioning state
- Irreversible lock after provisioning
- Debug port authentication before provisioning; disabled after

---

## 8. Requirements Trace

| Zone/Conduit | Requirement ID | Description |
| --- | --- | --- |
| Conduit C | REQ-SR-MFR-001 | Provisioning process access controls |
| Conduit C | REQ-SR-MFR-002 | Unique provisioning record |
| Zone 1 | REQ-SR-BOOT-001 | Firmware authenticity |
| Zone 1 | REQ-SR-BOOT-002 | Firmware integrity |
| Zone 1 | REQ-SR-ID-001 | Unique cryptographic identity |
| Zone 1 | REQ-SR-KEY-001 | Key protection |
| Zone 1 | REQ-SR-DBG-001 | Debug interface disabled in operational mode |
| Conduit A | REQ-SR-OTA-001 | Replay attack protection |
| Conduit A | REQ-SR-OTA-002 | Rollback protection |
