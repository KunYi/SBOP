# SBOP Data Flow

**Document ID:** SYS-DF-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines how data moves between components and across trust boundaries, with classification and protection requirements per flow.

---

## 2. Data Classification

| Data Type | Confidentiality | Integrity | Availability | Classification |
|-----------|----------------|-----------|--------------|----------------|
| Firmware Image | Low (binary public after release) | Critical (must be authentic) | High (must be available for OTA) | RESTRICTED |
| Image Signature | Low | Critical (enforces authenticity) | High | RESTRICTED |
| Root Key (KR) | Critical (compromise = all devices) | Critical | Medium (used only at provisioning) | SECRET |
| Device Key (KD) | Critical (device identity) | Critical | High (needed for mTLS) | SECRET |
| KD_Auth / KD_Debug / KD_Storage | Critical (derived from KD) | Critical | High | SECRET |
| UID | Low (useless without KR) | High (must not change) | High | INTERNAL |
| Boot Configuration (slot info, OTP state) | Low | Critical (controls boot) | High | INTERNAL |
| Telemetry Data | Medium (fleet behavior) | High (anomaly detection) | Medium | INTERNAL |
| Update Command | Low | Critical (controls OTA) | High | INTERNAL |

---

## 3. Data Flow Paths

### 3.1 Firmware Delivery Flow

```
Source Repo ──→ Build Pipeline ──→ HSM (sign) ──→ Firmware Registry ──→ Backend CDN ──→ Device
     │               │                 │                  │                  │              │
     │               │                 │                  │                  │              │
  [Commit      [Reproducible     [HSM signing]     [Registry          [TLS 1.3]      [Verify sig
   signing]     build, SBOM]                       integrity]                         + hash]
```

**Classification:** Integrity=Critical, Confidentiality=Low
**Protection:** Commit signing → reproducible build → HSM signature → registry integrity → TLS 1.3 transit → device re-verification

### 3.2 Device Authentication Flow

```
Device ──→ Backend
  │           │
  │  [KD_Auth client certificate, TLS 1.3 mutual auth]
  │           │
  │←── session_token ──│
```

**Classification:** Confidentiality=Critical (KD_Auth), Integrity=Critical
**Protection:** TLS 1.3 mutual auth, KD_Auth-derived certificate, session tokens time-limited (max 24h)

### 3.3 Key Provisioning Flow

```
TRNG (device) ──→ Station ──→ HSM ──→ Station ──→ Device Secure Element
                      │           │         │
                      │      HKDF(KR,UID)   │
                      │                     │
                   [encrypted channel, station attestation]
```

**Classification:** Confidentiality=Critical (KD in transit), Integrity=Critical
**Protection:** Station-to-device encrypted channel, station attestation verified before session, KD zeroized from station after injection

### 3.4 Telemetry Flow

```
Device ──→ Backend Telemetry Ingestion ──→ Anomaly Detection ──→ Dashboard
  │              │                              │
  │  [TLS 1.3, HMAC integrity, identity hashed]
```

**Classification:** Confidentiality=Medium, Integrity=High, Availability=Medium
**Protection:** TLS 1.3, device identity hashed (anonymized), HMAC integrity on telemetry records

### 3.5 Debug Flow

```
Debug Host ──→ Device Debug Port
                  │
           [Challenge-response auth, rate-limited]
                  │
           Zone 1 keys NOT accessible even after auth
```

**Classification:** Confidentiality=Critical (key material), Integrity=Critical
**Protection:** Debug lifecycle states, challenge-response authentication, rate limiting, KeyRef handles (never raw keys)

---

## 4. Trust Boundaries and Protection

| Boundary | Data Crossing | Protection Required |
|----------|--------------|---------------------|
| Network (Device ↔ Backend) | Firmware, telemetry, auth | TLS 1.3 + mutual auth |
| Network (Device ↔ Attacker) | All — assume intercepted | Crypto enforcement (verify before trust) |
| Device Zone 1 ↔ Zone 2 | Boot results, slot info | MPU/MMU isolation, read-only Zone 1 after boot |
| Secure Element ↔ Zone 1 | Key references (KeyRef) | Never raw keys; KeyRef handles only |
| Manufacturing Station ↔ Device | UID, KD | Encrypted channel, station attestation |
| Build Pipeline ↔ HSM | Image hash (to sign) | Air-gapped or network-isolated HSM |
| Developer ↔ Source Repo | Source code | Commit signing, branch protection |

---

## 5. Data Protection Summary

| Protection | Applied To |
|------------|------------|
| TLS 1.3 + mutual auth | All network communication |
| ECDSA / Ed25519 signature | Firmware images, attestation proofs |
| SHA-256 hash | Firmware images (integrity) |
| HKDF derivation | All keys below KR |
| Encrypted channel | Provisioning (station → device) |
| HMAC integrity | Telemetry records |
| Rate limiting | Debug authentication |
| Zeroization | KD on station after provisioning; all keys on tamper |

---

## 6. References

| Document | Reference |
|----------|-----------|
| System Context | `../00_Architecture/System_Context.md` |
| Trust Model | `../00_Architecture/Trust_Model.md` |
| Zone Conduit Model | `../04_Security/Zone_Conduit_Model.md` |
| Key Hierarchy | `Key_Hierarchy.md` |
| Image Format | `Image_Format.md` |
