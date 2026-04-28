# OTA State Machine Specification

**Document ID:** SUB-OTA-SM-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. States

State names aligned with `State_Machine.md`:

| State | Description |
|-------|-------------|
| IDLE | No update in progress |
| AUTHENTICATE | Mutual TLS with backend (KD_Auth client certificate) |
| CHECK | Poll backend for available firmware update |
| DOWNLOAD | Receive firmware image via TLS 1.3 |
| VERIFY | Device-side signature (ECDSA P-256/Ed25519) + SHA-256 hash verification |
| STAGE | Write verified image to inactive slot |
| ACTIVATE | Request boot activation; boot verifies and executes |
| ROLLBACK | Revert to previous valid image on boot failure |
| ERROR | Terminal error state; requires backend recovery command |

---

## 2. State Transitions

```
IDLE ─────────────────────────→ AUTHENTICATE
                                    │
                           (failure)│ (success)
                                    ▼
                                  CHECK
                                    │
                      (no update)   │ (update available)
                          ┌─────────┘
                          ▼
                        IDLE      DOWNLOAD
                                    │
                           (failure)│ (success)
                                    ▼
                                  VERIFY
                                    │
                           (failure)│ (success)
                          ┌─────────┘
                          ▼
                      DOWNLOAD    STAGE
                      (retry)       │
                           (failure)│ (success)
                                    ▼
                                 ACTIVATE
                                    │
                    ┌───────────────┼───────────────┐
                    │ (success)     │ (boot failure) │ (unrecoverable)
                    ▼               ▼                ▼
                  IDLE          ROLLBACK          ERROR
                                    │                │
                                    ▼                │
                                  IDLE ←─────────────┘
```

---

## 3. State Descriptions

### AUTHENTICATE

- Device presents KD_Auth-derived TLS client certificate
- Backend verifies certificate chain to KR
- Mutual TLS 1.3 session established
- Session token issued (time-limited)

Failure: → IDLE (retry later)
Error: → ERROR (after repeated failures)

### CHECK

- Device polls backend: `GET /firmware/check?uid=<UID>&version=<current>`
- Backend evaluates: device in group? version applicable? policy allows?
- Response: NO_UPDATE or UPDATE_AVAILABLE with firmware URL and metadata

Failure: → IDLE (no update available)
Error: → ERROR (after repeated failures)

### DOWNLOAD

- Firmware image downloaded via TLS 1.3
- Range requests supported for interruption recovery
- Image buffered in RAM (not written to flash until VERIFY passes)
- Download progress reported to backend via telemetry

Failure: → DOWNLOAD (retry with Range resume)
Error: → ERROR (after max retries)

### VERIFY

- Device independently re-verifies firmware (backend may be compromised):
  - Signature verification: ECDSA P-256 or Ed25519 against KI public key
  - SHA-256 hash over image body, constant-time comparison
- Backend authentication is NOT sufficient — device always re-verifies

Failure: → DOWNLOAD (discard image, re-download)
Error: → ERROR (after repeated verification failures)

### STAGE

- Verified image written to inactive slot
- Atomic write: slot marked PENDING only after full write + verify
- Active slot never modified during STAGE
- Flash write verified by read-back

Failure: → ERROR (flash write failure)

### ACTIVATE

- Request reboot; boot performs full verification chain (12 phases)
- Boot verifies PENDING slot: signature, integrity, version
- On success: PENDING → ACTIVE, previous ACTIVE → INACTIVE
- On failure: watchdog reset → boot selects other slot

No explicit transition back — boot handles slot selection on next reset.

### ROLLBACK

- Entered when new firmware fails to boot
- Boot selects previous ACTIVE slot
- Failed slot marked INVALID
- Rollback event reported to backend via telemetry

Transitions: → IDLE (after successful boot of previous image)

### ERROR

- Terminal state within OTA subsystem
- All OTA operations suspended
- Device reports ERROR state to backend
- Recovery: backend sends recovery command to reset OTA state

Transitions: → IDLE (on recovery command)

---

## 4. Invariants

| # | Invariant |
|---|-----------|
| I1 | No activation without full signature + integrity + version verification |
| I2 | Active slot is never overwritten during OTA |
| I3 | All verification failures route to DOWNLOAD (retry) or ERROR (terminal) |
| I4 | STAGE only proceeds after VERIFY passes |
| I5 | Boot performs independent verification before ACTIVATE commitment |

---

## 5. References

| Document | Reference |
|----------|-----------|
| OTA Overview | `OTA_Overview.md` |
| OTA Flow Pseudocode | `OTA_Flow_Pseudocode.md` |
| OTA Recovery | `OTA_Recovery.md` |
| State Machine | `../../02_System_Design/State_Machine.md` |
| Unified State Model | `../../02_System_Design/Unified_State_Model.md` |
