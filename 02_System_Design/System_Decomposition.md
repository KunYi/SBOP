# SBOP System Decomposition

**Document ID:** SYS-DEC-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the logical decomposition of the SBOP system into independent but interacting components, with explicit interfaces, responsibilities, and failure containment boundaries.

---

## 2. Design Principles

| Principle | Description |
|-----------|-------------|
| Hardware abstraction | No direct hardware dependency in core logic; platform abstraction layer |
| Separation of concerns | Each component has a single, well-defined responsibility |
| Explicit interfaces | All interactions are defined and constrained by formal contracts |
| Fail-safe design | All components must handle failure deterministically |
| Least privilege | Each component accesses only the resources and keys it needs |
| Defense in depth | Critical checks (signature, hash, version) are independently verified |

---

## 3. System Components

### 3.1 Boot Component

| Attribute | Detail |
|-----------|--------|
| Responsibility | Firmware verification, slot selection, anti-rollback, secure transition to application |
| Interfaces | Crypto (verify_sig, compute_hash), Identity (get_ki_public_key), Platform (otp_read/write) |
| Error domain | ERR-BOOT-* |
| Failure mode | FAILSAFE — no firmware execution |
| Criticality | MAXIMUM — any failure is catastrophic |

### 3.2 Update (OTA) Component

| Attribute | Detail |
|-----------|--------|
| Responsibility | Backend authentication, firmware download, device-side verification, image staging, rollback |
| Interfaces | Crypto (verify_sig, compute_hash), Identity (get_device_certificate), Boot (get_slot_info, request_activation), Backend (mTLS) |
| Error domain | ERR-OTA-* |
| Failure mode | Rollback to previous valid image |
| Criticality | HIGH — failure degrades to previous version |

### 3.3 Identity Component

| Attribute | Detail |
|-----------|--------|
| Responsibility | Device identity (UID + KD), provisioning, registration, anti-cloning, key lifecycle |
| Interfaces | Crypto (hkdf_derive), Backend (registration), Secure Element (key_store, key_get_ref) |
| Error domain | ERR-ID-* |
| Failure mode | Device untrusted — no backend operations |
| Criticality | MAXIMUM — device identity is foundational |

### 3.4 Crypto Component

| Attribute | Detail |
|-----------|--------|
| Responsibility | Signature verification, hash computation, key derivation, constant-time primitives |
| Interfaces | Boot (verify_sig, compute_hash), Identity (hkdf_derive), Platform (crypto engine) |
| Error domain | ERR-CRYPTO-* |
| Failure mode | Return error to caller; context determines handler |
| Criticality | MAXIMUM — all security depends on crypto correctness |

### 3.5 Backend Component

| Attribute | Detail |
|-----------|--------|
| Responsibility | Firmware registry, device registry, key management (KR in HSM), deployment control, telemetry |
| Interfaces | Device (mTLS), Build Pipeline (firmware upload), Operations (dashboard, alerts) |
| Error domain | Backend-side (out of SBOP firmware scope) |
| Failure mode | Device operates independently (caches auth token, verifies firmware locally) |
| Criticality | HIGH — compromise does not directly compromise devices (defense-in-depth) |

### 3.6 Toolchain Component

| Attribute | Detail |
|-----------|--------|
| Responsibility | Reproducible build, SBOM generation, HSM signing, firmware packaging |
| Interfaces | Source Repo (commit triggers), HSM (signing), Backend (firmware upload) |
| Error domain | Build-side (out of SBOP firmware scope) |
| Failure mode | Blocked release — device never receives bad firmware (verification at device) |
| Criticality | HIGH — supply chain is a primary attack vector |

---

## 4. Component Interaction Matrix

| From \ To | Boot | OTA | Identity | Crypto | Backend | Toolchain |
|-----------|------|-----|----------|--------|---------|-----------|
| Boot | — | slot info | get_ki_public | verify_sig, hash | — | — |
| OTA | slot_status | — | device_cert | verify_sig, hash | mTLS auth | — |
| Identity | — | — | — | hkdf_derive | register | — |
| Crypto | — | — | — | — | — | — |
| Backend | — | firmware URL | device registry | — | — | firmware upload |
| Toolchain | — | — | — | — | signed image | — |

---

## 5. Failure Containment

| Component Failure | Contained? | Impact |
|-------------------|------------|--------|
| Boot verification fails | Contained in bootloader | FAILSAFE — no Zone 2 execution |
| OTA download fails | Contained in OTA subsystem | Rollback to previous image |
| Identity corrupted | Contained in identity subsystem | Cannot authenticate to backend; factory re-provision |
| Crypto engine error | Contained in crypto subsystem | Error propagated to caller; handled per context |
| Backend unreachable | Contained — device autonomous | Device caches last known-good state; retries |
| Toolchain compromise | Contained — device re-verifies | Bad firmware rejected by device; never executed |

---

## 6. Security Boundary Diagram

```
┌──────────────────────────────────────────────────────┐
│                    Device Boundary                     │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │ Zone 1       │  │ Zone 1       │                  │
│  │ Boot         │  │ Crypto       │                  │
│  │ + Identity   │  │              │                  │
│  └──────┬───────┘  └──────┬───────┘                  │
│         │                 │                          │
│  ┌──────┴─────────────────┴───────┐                  │
│  │ Zone 2: OTA + Application      │                  │
│  │ (UNTRUSTED — cannot access     │                  │
│  │  Zone 1 memory or keys)        │                  │
│  └────────────────┬───────────────┘                  │
└───────────────────┼──────────────────────────────────┘
                    │ TLS 1.3 + Mutual Auth
┌───────────────────┴──────────────────────────────────┐
│                  Backend Boundary                      │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │ Firmware     │  │ Device       │                  │
│  │ Registry     │  │ Registry     │                  │
│  └──────────────┘  └──────────────┘                  │
│  ┌──────────────┐  ┌──────────────┐                  │
│  │ Key Manager  │  │ Deploy Ctrl  │                  │
│  │ (HSM)        │  │              │                  │
│  └──────────────┘  └──────────────┘                  │
└──────────────────────────────────────────────────────┘
```

---

## 7. References

| Document | Reference |
|----------|-----------|
| System Overview | `../00_Architecture/System_Overview.md` |
| Trust Model | `../00_Architecture/Trust_Model.md` |
| Zone Conduit Model | `../04_Security/Zone_Conduit_Model.md` |
| Interface Contracts | `../03_Subsystem/Interface_Contracts.md` |
| Data Flow | `Data_Flow.md` |
