# SBOP System State Machine

**Document ID:** SYS-SM-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines all valid system states and transitions, aligned with the 12-phase boot pseudocode and OTA state machine.

---

## 2. Boot State Machine

### 2.1 States

| State | Description |
|-------|-------------|
| RESET | Power-on / reset vector entry |
| INIT | MPU/MMU, clocks, crypto engine initialized |
| SELECT_SLOT | Choose highest-version valid slot (A or B) |
| LOAD_IMAGE | Load image header from selected slot |
| PARSE_HEADER | Parse and validate ImageHeader struct |
| VERIFY_SIGNATURE | Ed25519 signature check |
| VERIFY_INTEGRITY | SHA-256 hash over image body |
| CHECK_VERSION | Compare version against OTP counter |
| COMMIT_VERSION | Burn OTP counter if version > current |
| MARK_ACTIVE | Set slot ACTIVE flag |
| LOCK_BOOT | MPU/MMU lockdown, Zone 1 read-only |
| EXECUTE | Jump to Zone 2 entry point |
| FAILSAFE | Terminal state — no firmware execution |

### 2.2 Transitions

```
RESET
  │
  ▼
INIT ──────────────────────────────→ FAILSAFE (init failure)
  │
  ▼
SELECT_SLOT ────────────────────────→ FAILSAFE (no valid slot)
  │
  ▼
LOAD_IMAGE ─────────────────────────→ FAILSAFE (load failure)
  │
  ▼
PARSE_HEADER ───────────────────────→ FAILSAFE (invalid header)
  │
  ▼
VERIFY_SIGNATURE ───────────────────→ FAILSAFE (invalid signature)
  │
  ▼
VERIFY_INTEGRITY ───────────────────→ FAILSAFE (hash mismatch)
  │
  ▼
CHECK_VERSION ──────────────────────→ FAILSAFE (rollback detected)
  │
  ▼
COMMIT_VERSION ─────────────────────→ FAILSAFE (OTP write failure)
  │
  ▼
MARK_ACTIVE
  │
  ▼
LOCK_BOOT
  │
  ▼
EXECUTE
```

### 2.3 Slot Selection Sub-Machine

```
SELECT_SLOT
  │
  ├── Both valid, version_A > version_B  → Select A
  ├── Both valid, version_B > version_A  → Select B
  ├── Both valid, versions equal         → Use currently active
  ├── Only A valid                       → Select A
  ├── Only B valid                       → Select B
  └── Neither valid                      → FAILSAFE
```

---

## 3. OTA State Machine

### 3.1 States

| State | Description |
|-------|-------------|
| IDLE | No update in progress |
| AUTHENTICATE | Mutual TLS with backend |
| CHECK | Poll for available update |
| DOWNLOAD | Receive firmware image |
| VERIFY | Device-side signature + hash verification |
| STAGE | Write to inactive slot |
| ACTIVATE | Request boot activation |
| ROLLBACK | Revert to previous slot |
| ERROR | Terminal error, await recovery |

### 3.2 Transitions

```
IDLE
  │
  ▼
AUTHENTICATE ─────────────────────→ ERROR (auth failure)
  │
  ▼
CHECK ────────────────────────────→ IDLE (no update available)
  │
  ▼
DOWNLOAD ─────────────────────────→ ERROR (download failure, retryable)
  │
  ▼
VERIFY ───────────────────────────→ DOWNLOAD (verification failure, re-download)
  │
  ▼
STAGE ────────────────────────────→ ERROR (stage failure, retryable)
  │
  ▼
ACTIVATE ─────────────────────────→ IDLE (success)
  │                                │
  │                                └──→ ROLLBACK (boot failure)
  │                                       │
  └───────────────────────────────────────┘ (revert, then IDLE)
```

---

## 4. Identity Lifecycle State Machine

### 4.1 States

| State | Description | Allowed Operations |
|-------|-------------|---------------------|
| UNINITIALIZED | Factory-fresh, no identity | None |
| PROVISIONING | Identity being created | UID generation, KD injection |
| REGISTERED | Backend registration complete | Normal operations |
| LOCKED | Identity immutable, device operational | All normal operations |
| REVOKED | Backend-revoked (compromised) | None — all operations denied |
| TAMPER_LOCKED | Keys zeroized (tamper response) | None — permanently inoperative |

### 4.2 Transitions

```
UNINITIALIZED
  │
  ▼
PROVISIONING ──────────────────────→ UNINITIALIZED (retryable failure)
  │
  ▼
REGISTERED
  │
  ▼
LOCKED
  │
  ├──────────────────────────────→ REVOKED (backend revocation)
  │
  └──────────────────────────────→ TAMPER_LOCKED (tamper detected, permanent)
```

---

## 5. Unified System State Machine

Integrating Boot + OTA + Identity:

```
                    ┌──────────────────────────────┐
                    │      UNINITIALIZED            │
                    │   (factory-fresh)             │
                    └────────────┬─────────────────┘
                                 │ provision
                                 ▼
                    ┌──────────────────────────────┐
                    │      PROVISIONING             │
                    │   (identity creation)         │
                    └────────────┬─────────────────┘
                                 │ register
                                 ▼
              ┌──────────────────────────────────────┐
              │          REGISTERED / LOCKED          │
              │  ┌────────────────────────────────┐  │
              │  │  BOOT CYCLE                     │  │
              │  │  RESET → ... → EXECUTE          │  │
              │  │       ↓ (failure)               │  │
              │  │  FAILSAFE (await OTA)           │  │
              │  └────────────────────────────────┘  │
              │  ┌────────────────────────────────┐  │
              │  │  OTA CYCLE                      │  │
              │  │  IDLE → ... → ACTIVATE → IDLE   │  │
              │  │       ↓ (failure)                │  │
              │  │  ROLLBACK → IDLE                 │  │
              │  └────────────────────────────────┘  │
              └────────────┬─────────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
   ┌──────────────────┐    ┌──────────────────────┐
   │     REVOKED       │    │   TAMPER_LOCKED       │
   │  (backend order)  │    │  (keys zeroized)      │
   └──────────────────┘    └──────────────────────┘
```

---

## 6. Invalid Transitions

Any undefined transition shall result in:
- Boot context: → FAILSAFE
- OTA context: → ERROR (with rollback path)
- Identity context: → REJECT (no state change)

---

## 7. References

| Document | Reference |
|----------|-----------|
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| Boot State Detail | `../03_Subsystem/Boot/Boot_State_Detail.md` |
| OTA State Machine | `../03_Subsystem/Update/OTA_State_Machine.md` |
| Identity Lifecycle | `../03_Subsystem/Identity/Identity_Overview.md` |
| Unified State Model | `Unified_State_Model.md` |
