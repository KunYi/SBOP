# Anti-Cloning Strategy

**Document ID:** SUB-ID-AC-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines mechanisms to prevent duplication of device identity and detect cloning attempts.

---

## 2. Threat Model

| Threat | Description | Attacker Capability |
|--------|-------------|---------------------|
| Firmware copying | Clone firmware image to another device | Physical access, debug tools |
| Key extraction | Extract KD from secure element | Chip decapsulation, side-channel (SL 3+) |
| UID spoofing | Emulate another device's UID | Knowledge of UID generation |
| Provisioning replay | Replay provisioning protocol to register clone | Network access during provisioning |
| Supply chain clone | Introduce clone during manufacturing | Insider, supply chain interdiction |

---

## 3. Protection Mechanisms

### 3.1 Hardware Binding

Identity is bound to the physical device via one of:

| Method | Binding Strength | Clone Resistance |
|--------|-----------------|------------------|
| TRNG-only | Moderate — UID is random, KD injected | Attacker with UID cannot derive KD without KR |
| SRAM PUF | Strong — identity implicit in silicon | Cloning requires identical silicon; statistically infeasible |
| Hardware Unique Key (HUK) | Very strong — burned at fab | Requires silicon-level forgery |

**PUF Integration (Preferred):**

When SRAM PUF is available:
- UID = PUF_Enroll(startup_values) — derived from PUF response, not stored
- KD = HKDF(KR, UID) — derived internally after PUF reconstruction
- Identity is never stored in flash; it emerges from silicon on each boot
- Cloning requires physically identical silicon with identical PUF characteristics → infeasible

### 3.2 Key Derivation Binding

```
KR (HSM, air-gapped)  ──→  KD = HKDF(KR, UID)  ──→  KD_Auth, KD_Debug, KD_Storage
```

- Copying UID alone is useless without KR
- KR never leaves HSM; KD derived only during provisioning
- KD stored in secure element / TEE, accessed via KeyRef handles only
- KD never exported in raw form after injection

### 3.3 Backend Deduplication

Backend enforces identity uniqueness:

| Check | Mechanism | Action on Violation |
|-------|-----------|---------------------|
| Duplicate UID | UID already registered | Reject registration, flag as clone attempt |
| Impossible geo-movement | Same UID from geographically distant IPs in short time | Flag anomaly, require re-authentication |
| Batch anomaly | Multiple UIDs from unexpected batch range | Flag batch, hold registrations for review |
| Certificate mismatch | KD_Auth certificate does not chain to KR | Reject connection, revoke if repeated |

### 3.4 Secure Provisioning

Trust established during provisioning (see `Provisioning_Flow.md`):

1. Station attestation verified before each session
2. Encrypted channel between station and device
3. KD injected once, then zeroized from station
4. Device locks identity (irreversible) after registration
5. Backend verifies: UID unique, batch authorized, station authorized

### 3.5 Enrollment Protocol

```
Device                                Station                           Backend
  │                                      │                                  │
  │──── UID (TRNG) ─────────────────────→│                                  │
  │                                      │──── UID, batch_id ──────────────→│
  │                                      │                                  │──── Validate batch
  │                                      │←──── KD = HKDF(KR, UID) ────────│
  │←──── KD (encrypted channel) ────────│                                  │
  │                                      │                                  │
  │──── Sign(KD, UID || nonce) ────────→│                                  │
  │                                      │──── attestation_proof ──────────→│
  │                                      │                                  │──── Verify proof
  │                                      │←──── registration_ack ───────────│
  │←──── LOCK_IDENTITY ─────────────────│                                  │
  │                                      │                                  │
  │                                      │──── zeroize KD ─────────────────→│
```

---

## 4. Detection

| Layer | Mechanism | Trigger |
|-------|-----------|---------|
| Backend dedup | UID uniqueness check | Duplicate UID at registration |
| Anomaly detection | Geo-impossible movement, rate anomalies | Same UID from different IPs within threshold |
| Certificate monitoring | Validate KD_Auth certificate chain | Certificate not chaining to KR |
| Fleet consistency | Batch range validation, version distribution | Unexpected batch patterns |
| PUF health | PUF reliability monitoring (if PUF-based) | PUF response degradation over time |

---

## 5. Response

| Event | Response |
|-------|----------|
| Duplicate registration attempt | Reject, log, flag for security review |
| Confirmed clone | Revoke device identity at backend; add UID to revocation list |
| Anomaly (unconfirmed) | Require re-authentication; escalate if repeated |
| Supply chain clone detected | Revoke entire batch range; trigger incident response |
| PUF degradation | Schedule device for re-enrollment or decommissioning |

---

## 6. References

| Document | Reference |
|----------|-----------|
| Provisioning Flow | `Provisioning_Flow.md` |
| Identity Overview | `Identity_Overview.md` |
| Identity Failure Model | `Identity_Failure_Model.md` |
| Manufacturing Security | `../../04_Security/Manufacturing_Security.md` |
| Trust Model | `../../00_Architecture/Trust_Model.md` |
