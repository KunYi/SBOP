# Key Management Operations

**Document ID:** OPS-KEY-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the lifecycle management of all cryptographic keys in the SBOP ecosystem: generation, storage, distribution, rotation, revocation, and destruction.

---

## 2. Key Hierarchy

```
Root Key (KR) ──── HKDF ──→ Device Key (KD)
       │                          │
       │                          ├──→ KD_Auth (OTA mutual TLS)
       │                          ├──→ KD_Debug (debug challenge-response)
       │                          └──→ KD_Storage (optional flash encryption)
       │
       └──→ Image Signing Key (KI) ──→ signs firmware images
```

### 2.1 Key Type Registry

| Key | Type | Generation | Storage | Lifetime | Rotation |
| --- | --- | --- | --- | --- | --- |
| KR | Symmetric (256-bit) | HSM TRNG | HSM (FIPS 140-2 L3) | Permanent | Emergency only |
| KD | Symmetric (256-bit) | HKDF(KR, UID) | Secure element / TEE | Device lifetime | Per-device via KR rotation |
| KD_Auth | Symmetric (256-bit) | HKDF(KD, "AUTH") | Secure element | Session-derived | Per session |
| KD_Debug | Symmetric (256-bit) | HKDF(KD, "DEBUG") | Secure element | Device lifetime | Per-device via KR rotation |
| KD_Storage | Symmetric (256-bit) | HKDF(KD, "STORAGE") | Secure element | Device lifetime | Per-device via KR rotation |
| KI (Ed25519) | Asymmetric key pair | HSM TRNG | HSM | Per firmware version | Per release |

---

## 3. Key Storage Requirements

### 3.1 Production Keys

| Location | Protection | Access Control |
| --- | --- | --- |
| HSM (KR, KI) | FIPS 140-2 Level 3+ | Multi-party quorum (N-of-M) |
| Secure element (KD, sub-keys) | Hardware isolation | Key API only (no raw export) |
| TEE (KD, sub-keys) | TrustZone / PMP isolation | Key handle only (no raw export) |

### 3.2 Key Access Matrix

| Actor | KR | KI (private) | KI (public) | KD | KD_Debug |
| --- | --- | --- | --- | --- | --- |
| Signing operator | No direct access | Sign-only via HSM | Read | No access | No access |
| Build system | No access | Sign-only via HSM API | Read | No access | No access |
| Device firmware (Zone 1) | No access | Read (verify key) | Read | Key handle only | Key handle only |
| Device firmware (Zone 2) | No access | No access | No access | No access | No access |
| Backend server | No access | No access | Read | Derived per-device | Derived on demand |
| Debug session | No access | No access | No access | No access | Derived for auth |
| Provisioning station | No access | No access | Read | Temporary (provisioning only) | Temporary (provisioning only) |

---

## 4. Key Generation Ceremony

### 4.1 Root Key (KR) Generation

```
Prerequisites:
  - FIPS 140-2 Level 3 HSM initialized and attested
  - Minimum 3 authorized operators present
  - Ceremony script reviewed and approved
  - Video recording enabled
  - Air-gapped environment confirmed

Procedure:
  1. Operators authenticate to HSM (each presents credential)
  2. Quorum confirmed (≥ 2 of 3 operators)
  3. HSM generates KR: TRNG → 256-bit random
  4. KR never exported — key handle stored in HSM
  5. KR public commitment generated: HMAC(KR, "COMMIT") → stored off-HSM
  6. Backup: HSM performs M-of-N key split to smart cards
  7. Each operator receives one share in tamper-evident envelope
  8. Envelopes stored in separate physical safes
  9. Ceremony log signed by all operators
  10. Video recording archived
```

### 4.2 Image Signing Key (KI) Generation

```
Prerequisites:
  - KR ceremony complete and verified
  - New KI generation authorized by security architect

Procedure:
  1. Operators authenticate to HSM (quorum required)
  2. HSM generates KI key pair (Ed25519)
  3. KI private key never exported
  4. KI public key exported for distribution:
     - Embedded in bootloader (Zone 1) at build time
     - Published in backend firmware registry
     - Included in SBOM
  5. KI key handle stored, linked to firmware version range
```

---

## 5. Key Rotation

### 5.1 Planned Rotation

| Key | Rotation Trigger | Procedure |
| --- | --- | --- |
| KI | Each firmware release (or per N releases) | New KI generated; public key embedded in new bootloader or image |
| KR | Every 5 years or after security incident | New KR generated; all KDs re-derived; all devices re-provisioned or updated |

### 5.2 Emergency Rotation

Triggered by:
- Suspected key compromise
- Insider departure with key access
- HSM security alert
- Cryptographic algorithm deprecation

Procedure:
1. Security architect declares key compromise incident
2. Affected key immediately revoked in backend
3. New key generated via emergency ceremony (same quorum, accelerated timeline)
4. All devices forced to accept new key via emergency OTA (signed with old key while valid + new key)
5. Old key destroyed in HSM after transition window
6. Incident report filed

### 5.3 Key Transition Window

```
Time  ──────────────────────────────────────────→
      │                    │                    │
      KI_v1 active         KI_v1 + KI_v2        KI_v2 active
      (v1 signs)           (both accepted)      (v2 signs)
                           ← Transition window →
```

Devices accept firmware signed by either KI_v1 or KI_v2 during the transition window. After the window closes, KI_v1 signatures are rejected.

---

## 6. Key Revocation

### 6.1 Revocation Process

| Step | Action | System |
| --- | --- | --- |
| 1 | Mark key as REVOKED in HSM | HSM |
| 2 | Add key hash to backend revocation list | Backend |
| 3 | Notify all devices of revocation (OTA or telemetry channel) | Backend → Device |
| 4 | Devices cache revocation list in secure storage | Device |
| 5 | Boot verification rejects images signed by revoked key | Device |
| 6 | Revocation event logged with timestamp and reason | All |

### 6.2 Revocation Scope

| Key Revoked | Impact |
| --- | --- |
| KI (single version) | Only firmware signed with that KI rejected; other versions unaffected |
| KI (all versions) | All firmware must be re-signed with new KI |
| KR | All device keys invalidated; full re-provisioning required (catastrophic) |

---

## 7. Key Destruction

### 7.1 Device-Side (Tamper Response)

```
On TAMPER_CRITICAL:
  1. Overwrite KD in secure element (multiple passes)
  2. Overwrite all derived keys (KD_Auth, KD_Debug, KD_Storage)
  3. Verify zeroization: read back, must be all-zero or random
  4. Set device state to TAMPER_LOCK (irreversible)
  5. Log tamper event to tamper log (if still writable)
```

### 7.2 Server-Side (Key Retirement)

```
On key retirement:
  1. Export HSM audit log for the key
  2. Delete key from HSM (cryptographic erase)
  3. Verify deletion: attempt operation with key handle → must fail
  4. Destroy backup shares (physical shredding of smart cards)
  5. Log key destruction with witness signatures
```

---

## 8. Audit Requirements

| Event | Logged Data | Retention |
| --- | --- | --- |
| Key generation | Timestamp, key ID, algorithm, operators present, ceremony log | Permanent |
| Signing operation | Timestamp, key ID, firmware hash, operator identity, HSM serial | 10 years |
| Key rotation | Timestamp, old key ID, new key ID, reason | 10 years |
| Key revocation | Timestamp, key ID, reason, authorizing officer | Permanent |
| Key destruction | Timestamp, key ID, method, witnesses | Permanent |
| Access attempt (denied) | Timestamp, key ID, requesting identity, reason denied | 5 years |

---

## 9. References

| Document | Reference |
| --- | --- |
| Key Hierarchy | `../02_System_Design/Key_Hierarchy.md` |
| Supply Chain Security | `../04_Security/Supply_Chain_Security.md` §4 |
| Manufacturing Security | `../04_Security/Manufacturing_Security.md` §4 |
| Architecture Decision Record | `../00_Architecture/Architecture_Decision_Record.md` (ADR-006, ADR-007) |
| Incident Response | `Incident_Response.md` |
