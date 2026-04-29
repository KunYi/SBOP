# Unified State Model

**Document ID:** SYS-USM-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines a unified system state machine integrating Boot (12 phases), OTA (8 states), and Identity lifecycle (6 states). This model is the single source of truth for all system state transitions.

---

## 2. Boot State Definitions

Aligned with `State_Machine.md` and `Boot_Flow_Pseudocode.md`:

| State | Description |
|-------|-------------|
| BOOT_RESET | Power-on / reset vector entry |
| BOOT_INIT | Initialize MPU/MMU, clocks, crypto engine, TRNG health check |
| BOOT_SELECT_SLOT | Select highest-version valid slot (A or B) |
| BOOT_LOAD_IMAGE | Load image header from selected slot |
| BOOT_PARSE_HEADER | Parse and validate ImageHeader struct |
| BOOT_OTA_DECRYPT | ECDH + AES-256-GCM decrypt + KD_Storage re-encrypt (if FLAG_OTA_PENDING) |
| BOOT_VERIFY_SIGNATURE | Ed25519 signature verification |
| BOOT_VERIFY_INTEGRITY | SHA-256 hash over image body, constant-time compare |
| BOOT_CHECK_VERSION | Compare version against version counter |
| BOOT_COMMIT_VERSION | Write version counter (if version > current) |
| BOOT_MARK_TESTING | Set slot TESTING (test/confirm model) |
| BOOT_LOCK_BOOT | MPU/MMU lockdown, Zone 1 read-only |
| BOOT_EXECUTE | Jump to Zone 2 entry point |
| BOOT_FAILSAFE | Terminal — no firmware execution |

---

## 3. OTA State Definitions

| State | Description |
|-------|-------------|
| OTA_IDLE | No update in progress |
| OTA_AUTHENTICATE | Mutual TLS with backend |
| OTA_CHECK | Poll backend for available update |
| OTA_DOWNLOAD | Receive firmware image |
| OTA_VERIFY | Device-side signature + hash verification |
| OTA_STAGE | Write to inactive slot |
| OTA_ACTIVATE | Request boot activation |
| OTA_ROLLBACK | Revert to previous slot |
| OTA_ERROR | Terminal error, await recovery command |

---

## 4. Identity State Definitions

| State | Description |
|-------|-------------|
| ID_UNINITIALIZED | Factory-fresh, no identity |
| ID_PROVISIONING | Identity being created |
| ID_REGISTERED | Backend registration complete |
| ID_LOCKED | Identity immutable, operational |
| ID_REVOKED | Backend-revoked (compromised) |
| ID_TAMPER_LOCKED | Keys zeroized (permanent) |

---

## 5. State Transitions

### 5.1 Boot Flow

```
BOOT_RESET → BOOT_INIT → BOOT_SELECT_SLOT → BOOT_LOAD_IMAGE → BOOT_PARSE_HEADER
                                                                      │
                                                                      ▼
                                                               BOOT_OTA_DECRYPT
                                                                      │
                                                                      ▼
                                                               BOOT_VERIFY_SIGNATURE
                                                                      │
                                                                      ▼
                                                               BOOT_VERIFY_INTEGRITY
                                                                      │
                                                                      ▼
                                                               BOOT_CHECK_VERSION
                                                                      │
                                                                      ▼
                                                               BOOT_COMMIT_VERSION
                                                                      │
                                                                      ▼
                                                               BOOT_MARK_TESTING
                                                                      │
                                                                      ▼
                                                               BOOT_LOCK_BOOT
                                                                      │
                                                                      ▼
                                                               BOOT_EXECUTE

Any phase failure → BOOT_FAILSAFE
```

### 5.2 OTA Flow

```
OTA_IDLE → OTA_AUTHENTICATE → OTA_CHECK → OTA_DOWNLOAD → OTA_VERIFY
                                                               │
                                              (failure) ──────┤
                                                               │ (success)
                                                               ▼
                                                          OTA_STAGE
                                                               │
                                                               ▼
                                                          OTA_ACTIVATE
                                                               │
                                              ┌────────────────┼────────────────┐
                                              │ (success)      │ (boot failure)  │
                                              ▼                ▼                 │
                                         BOOT_EXECUTE    OTA_ROLLBACK           │
                                                              │                  │
                                                              └──→ OTA_IDLE ────┘

Any unrecoverable failure → OTA_ERROR
```

### 5.3 Identity Lifecycle

```
ID_UNINITIALIZED → ID_PROVISIONING → ID_REGISTERED → ID_LOCKED
                                                           │
                                        ┌──────────────────┼──────────────────┐
                                        │ (revocation)     │ (tamper detect)   │
                                        ▼                  ▼
                                   ID_REVOKED        ID_TAMPER_LOCKED
```

---

## 6. Cross-Phase Transitions

| From | To | Condition |
|------|----|-----------|
| BOOT_EXECUTE | OTA_IDLE | Application initiates OTA |
| BOOT_FAILSAFE | OTA_IDLE | Recovery firmware download |
| OTA_ROLLBACK | BOOT_RESET | Reboot to previous image |
| ID_LOCKED | BOOT_RESET | Normal device operation (after provisioning) |

---

## 7. Security Invariants

| # | Invariant | Enforced By |
|---|-----------|-------------|
| I1 | No EXECUTE without passing all 12 boot phases | Boot state machine |
| I2 | No state bypass possible | Sequential phase enforcement |
| I3 | All verification failures → FAILSAFE | Each phase has FAILSAFE edge |
| I4 | OTA firmware must be verified before activation | OTA_VERIFY → OTA_STAGE gate |
| I5 | Only one ACTIVE firmware image | Slot selection logic |
| I6 | Identity lock is irreversible | Hardware fuse + OTP |
| I7 | Tamper response is permanent | Key zeroization + fuse |
| I8 | Zone 2 cannot access Zone 1 after LOCK_BOOT | MPU/MMU configuration |

---

## 8. Formal Verification Targets

This model is used to verify:

- **P1 (Safety):** No path reaches BOOT_EXECUTE without passing all verification states
- **P2 (Safety):** All failure paths terminate in BOOT_FAILSAFE, OTA_ERROR, or OTA_ROLLBACK
- **P3 (Liveness):** If a valid image exists, boot eventually reaches BOOT_EXECUTE
- **P4 (Safety):** ID_LOCKED → ID_UNINITIALIZED is unreachable (lock is irreversible)
- **P5 (Safety):** ID_TAMPER_LOCKED → any other state is unreachable

---

## 9. Authority Rule

This document is the authoritative definition of all system states and transitions. All subsystem state definitions MUST conform to this model.

---

## 10. References

| Document | Reference |
|----------|-----------|
| State Machine | `State_Machine.md` |
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| OTA State Machine | `../03_Subsystem/Update/OTA_State_Machine.md` |
| Identity Overview | `../03_Subsystem/Identity/Identity_Overview.md` |
