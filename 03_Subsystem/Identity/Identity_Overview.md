# SBOP Identity Subsystem Specification

**Document ID:** SUB-ID-OV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the Identity subsystem responsible for establishing, maintaining, and verifying device identity. Device identity is the foundation for backend trust, OTA authorization, anti-cloning, and fleet management.

---

## 2. Scope

**Includes:**
- Device identity definition (UID + KD)
- Provisioning process (factory identity creation)
- Trust establishment with backend (registration, authentication)
- Anti-cloning mechanisms (hardware binding, backend dedup)
- Identity lifecycle management (UNINITIALIZED → LOCKED → REVOKED)

**Excludes:**
- Firmware verification (Boot subsystem)
- OTA orchestration (Update subsystem)
- Key ceremony and HSM operations (Operations)

---

## 3. Identity Model

### 3.1 Core Identity

Each device possesses a unique cryptographic identity:

| Element | Source | Size | Purpose |
| --- | --- | --- | --- |
| UID | TRNG at provisioning | 128 bits | Unique device identifier |
| KD | HKDF(KR, UID) | 256 bits | Device master key |
| KD_Auth | HKDF(KD, "AUTH") | 256 bits | OTA mutual TLS |
| KD_Debug | HKDF(KD, "DEBUG") | 256 bits | Debug authentication |
| KD_Storage | HKDF(KD, "STORAGE") | 256 bits | Flash encryption (optional) |

### 3.2 Identity Binding Options

| Method | Description | Platform Requirement |
| --- | --- | --- |
| TRNG-only | UID from TRNG, KD injected | TRNG (minimum) |
| PUF | UID derived from PUF response, KD derived internally | SRAM PUF or similar |
| HUK | Hardware Unique Key burned at silicon level | Secure element with HUK |

PUF is preferred when available: the device identity is implicit in the silicon, making extraction and cloning significantly harder.

---

## 4. Responsibilities

The Identity subsystem shall:

| # | Responsibility | Implementation |
| --- | --- | --- |
| 1 | Ensure each device has a unique identity | TRNG UID (128-bit) + backend dedup |
| 2 | Establish root trust during provisioning | KD = HKDF(KR, UID) binding |
| 3 | Bind device identity to backend | Registration with attestation proof |
| 4 | Prevent identity duplication | Hardware binding + backend dedup + anomaly detection |
| 5 | Protect identity from extraction | Secure element / TEE; KeyRef handles only |
| 6 | Support identity verification | Challenge-response with KD-derived keys |
| 7 | Enable secure decommissioning | Key zeroization on tamper; backend revocation |

---

## 5. Identity Lifecycle

```
UNINITIALIZED ──→ PROVISIONING ──→ REGISTERED ──→ LOCKED
                                                     │
                                                     ├──→ REVOKED (backend revocation)
                                                     └──→ TAMPER_LOCKED (keys zeroized, permanent)
```

| State | Description | Allowed Operations |
| --- | --- | --- |
| UNINITIALIZED | Factory-fresh device | None (no identity) |
| PROVISIONING | Identity being created | UID generation, key injection |
| REGISTERED | Backend registration complete | Normal operations |
| LOCKED | Identity immutable | All normal operations |
| REVOKED | Backend-revoked (compromised) | None — all operations denied |
| TAMPER_LOCKED | Tamper response (keys gone) | None — permanently inoperative |

---

## 6. Trust Establishment

Trust is established during provisioning (see `Provisioning_Flow.md`):

1. Device generates UID (TRNG)
2. Station derives KD = HKDF(KR, UID) using HSM
3. KD injected into device secure element
4. Device self-tests: signature verification, hash, TRNG health
5. Device generates attestation proof: `Sign(KD, UID || nonce)`
6. Backend validates: UID unique, batch authorized, station authorized
7. Device locks identity (irreversible)
8. Backend registers device in device registry

Trust is verified operationally (OTA authentication):
1. Device presents KD_Auth-derived client certificate
2. Backend verifies certificate chains to KR
3. Mutual TLS session established
4. Backend authorizes device for firmware updates

---

## 7. Anti-Cloning Strategy

| Layer | Mechanism |
| --- | --- |
| Hardware binding | UID from TRNG (or PUF/HUK) — cannot be copied off-device |
| Key derivation | KD = HKDF(KR, UID) — copying UID without KR is useless |
| Backend dedup | Backend rejects second registration of same UID |
| Anomaly detection | Backend flags: same UID from different IPs, impossible geographic movement |
| Secure element | KD stored in hardware-isolated storage — cannot be extracted |

---

## 8. References

| Document | Reference |
| --- | --- |
| Provisioning Flow | `Provisioning_Flow.md` |
| Identity Interface | `Identity_Interface.md` |
| Anti-Cloning | `Anti_Cloning.md` |
| Identity Failure Model | `Identity_Failure_Model.md` |
| Manufacturing Security | `../../04_Security/Manufacturing_Security.md` |
| Key Derivation | `../Crypto/Key_Derivation.md` |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-010) |
