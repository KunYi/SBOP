# SBOP System Flow

**Document ID:** SYS-FLOW-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Overview

Defines the end-to-end operational flow of the system from power-on to runtime update, aligned with the 12-phase boot pseudocode and OTA state machine.

---

## 2. Boot Flow (12 Phases)

Aligned with `Boot_Flow_Pseudocode.md`:

| Phase | State | Action | Key Verification |
|-------|-------|--------|------------------|
| 1 | RESET | Power-on, reset vector entry | — |
| 2 | INIT | Initialize MPU/MMU, clocks, crypto engine, TRNG health check | TRNG health |
| 3 | SELECT_SLOT | Select highest-version valid slot (A/B) | Slot metadata valid |
| 4 | LOAD_IMAGE | Load image header from selected slot | Magic number |
| 5 | PARSE_HEADER | Parse ImageHeader struct, validate fields | Header integrity |
| 6 | VERIFY_SIGNATURE | ECDSA P-256 / Ed25519 signature check | Signature valid |
| 7 | VERIFY_INTEGRITY | SHA-256 hash over image body, constant-time compare | Hash matches |
| 8 | CHECK_VERSION | Compare image version against OTP counter | Version >= OTP |
| 9 | COMMIT_VERSION | Burn OTP counter (only if version > current) | OTP write success |
| 10 | MARK_ACTIVE | Set slot ACTIVE, lock Zone 1 memory | Slot marked |
| 11 | LOCK_BOOT | MPU/MMU lockdown, Zone 1 read-only | MPU configured |
| 12 | EXECUTE | Jump to Zone 2 entry point | — |

Any verification failure → FAILSAFE (no execution, no information leak).

---

## 3. OTA Flow (7 Steps)

Aligned with `OTA_Flow_Pseudocode.md`:

| Step | State | Action | Protection |
|------|-------|--------|------------|
| 1 | AUTHENTICATE | Mutual TLS with KD_Auth client certificate | TLS 1.3, time-limited session token |
| 2 | CHECK | Poll backend for available update | Server certificate validation |
| 3 | DOWNLOAD | Receive firmware image to buffer | TLS 1.3, Range request support for resume |
| 4 | VERIFY | Device-side signature + hash verification | Same verification as boot (steps 6-7) |
| 5 | STAGE | Write verified image to inactive slot | Atomic slot write |
| 6 | ACTIVATE | Request boot switch to new image | Boot performs full verification chain |
| 7 | ROLLBACK | On boot failure, revert to previous slot | Previous valid image preserved |

---

## 4. Provisioning Flow (8 Steps)

Aligned with `Provisioning_Flow.md`:

| Step | Action | Protection |
|------|--------|------------|
| 1 | Device generates UID (TRNG) | TRNG health check |
| 2 | Station attests to backend | Station certificate verification |
| 3 | Station derives KD = HKDF(KR, UID) via HSM | HSM derive-only, no raw KR export |
| 4 | KD injected into device secure element | Encrypted channel, station attestation active |
| 5 | Device self-tests | Signature, hash, TRNG verification |
| 6 | Device generates attestation proof | Sign(KD, UID || nonce) |
| 7 | Backend validates and registers | UID uniqueness, batch authorization |
| 8 | Device locks identity (irreversible), station zeroizes KD | Irreversible state transition |

---

## 5. Failure Handling Flow

| Scenario | Detection | Response | Recovery |
|----------|-----------|----------|----------|
| Boot verification failure | Any phase 2-12 check fails | FAILSAFE — no execution | Requires valid firmware update |
| OTA download interrupted | Connection timeout | Retry with Range request | Resume from last byte received |
| OTA verification failure | Signature or hash mismatch | Discard image, retry | Re-download |
| OTA stage failure | Flash write error | Mark slot invalid | Retry or wait for next update |
| New image boot failure | Watchdog timeout after EXECUTE | Boot selects other slot | Automatic rollback |
| Provisioning failure | Any step 1-7 fails | Device remains UNINITIALIZED | Retry provisioning |

---

## 6. Trust Flow

```
Provisioning (KR → KD) → Registration (Backend trust) → Boot (KI verification) → OTA (KD_Auth mTLS) → Execute
```

- Root trust originates from provisioning (KR in HSM)
- Trust propagates through key hierarchy (KR → KD → KD_Auth/KD_Debug/KD_Storage)
- All firmware must chain back to KI → KR trust root
- Every operation requires cryptographic proof — no implicit trust

---

## 7. References

| Document | Reference |
|----------|-----------|
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| OTA Flow Pseudocode | `../03_Subsystem/Update/OTA_Flow_Pseudocode.md` |
| Provisioning Flow | `../03_Subsystem/Identity/Provisioning_Flow.md` |
| State Machine | `State_Machine.md` |
| Trust Model | `../00_Architecture/Trust_Model.md` |
