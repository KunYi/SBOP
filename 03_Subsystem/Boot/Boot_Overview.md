# SBOP Boot Subsystem Specification

**Document ID:** SUB-BOOT-OV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the Boot subsystem responsible for enforcing the root of trust at device startup. The Boot subsystem ensures that only authenticated and authorized firmware is executed.

---

## 2. Scope

**Includes:**
- Firmware verification (authenticity and integrity)
- Version validation (rollback protection)
- Slot selection (A/B dual-image model)
- Secure transition to application firmware
- FAILSAFE handling

**Excludes:**
- Firmware update logic (Update subsystem)
- Key provisioning (Identity subsystem)
- Hardware-specific drivers (platform layer)

---

## 3. Subsystem Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Boot Subsystem                      │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐ │
│  │ Slot     │   │ Signature│   │ Version          │ │
│  │ Selector │──→│ Verifier │──→│ Checker          │ │
│  └──────────┘   └──────────┘   └────────┬─────────┘ │
│                                         │            │
│            ┌────────────────────────────┘            │
│            ▼                                         │
│  ┌─────────────────┐   ┌────────────────┐           │
│  │ Integrity        │   │ FAILSAFE       │           │
│  │ Verifier         │──→│ Handler         │           │
│  └────────┬────────┘   └────────────────┘           │
│           │                                          │
│  ┌────────▼────────┐                                │
│  │ Image Activator  │                                │
│  │ (MARK_ACTIVE)    │                                │
│  └────────┬────────┘                                │
│           │                                          │
│  ┌────────▼────────┐                                │
│  │ Zone 2 Entry     │                                │
│  │ (EXECUTE)        │                                │
│  └─────────────────┘                                │
│                                                       │
└─────────────────────────────────────────────────────┘
```

---

## 4. Boot Flow (12 Phases)

| Phase | State | Description | On Failure |
|-------|-------|-------------|------------|
| 1 | RESET | Power-on / reset vector entry | — |
| 2 | INIT | Initialize MPU/MMU, clocks, crypto engine | FAILSAFE |
| 3 | SELECT_SLOT | Select highest-version valid slot (A or B) | Try other slot |
| 4 | LOAD_IMAGE | Load image header from selected slot | Try other slot |
| 5 | PARSE_HEADER | Parse ImageHeader struct, validate magic | FAILSAFE |
| 6 | VERIFY_SIGNATURE | Ed25519 signature check | FAILSAFE |
| 7 | VERIFY_INTEGRITY | SHA-256 hash over image body | FAILSAFE |
| 8 | CHECK_VERSION | Monotonic OTP counter comparison | FAILSAFE |
| 9 | COMMIT_VERSION | Burn OTP counter (only if version > current) | Retry then FAILSAFE |
| 10 | MARK_ACTIVE | Set slot ACTIVE, lock Zone 1 memory | — |
| 11 | LOCK_BOOT | MPU/MMU lockdown, Zone 1 read-only | — |
| 12 | EXECUTE | Jump to Zone 2 entry point | — |

---

## 5. Slot Selection Logic

```
if slot_A.valid and slot_B.valid:
    select slot with higher version
    if versions equal: select currently marked active

elif slot_A.valid and not slot_B.valid:
    select slot_A

elif not slot_A.valid and slot_B.valid:
    select slot_B

else:
    FAILSAFE  (no valid image)
```

A slot is "valid" when: signature OK, integrity OK, version >= OTP counter.

---

## 6. Key Interfaces

| Interface | Function | Direction | Error Codes |
|-----------|----------|-----------|-------------|
| Crypto | `verify_signature(image, sig, pubkey)` | Call | ERR-BOOT-CRYPTO-001 |
| Crypto | `compute_hash(image)` | Call | ERR-BOOT-CRYPTO-002 |
| Crypto | `constant_time_compare(a, b)` | Call | — |
| Identity | `get_device_identity()` | Call | ERR-ID-PROV-001 |
| Identity | `get_ki_public_key()` | Call | ERR-BOOT-CRYPTO-004 |
| Platform | `otp_read(addr)` | Call | ERR-STOR-READ-001 |
| Platform | `otp_write(addr, val)` | Call | ERR-STOR-WRITE-001 |
| Update | `get_slot_info(slot)` | Call | ERR-STOR-READ-001 |
| Update | `mark_active(slot)` | Call | ERR-STOR-WRITE-001 |

---

## 7. Safety Design

| Guarantee | Mechanism |
|-----------|-----------|
| No boot of compromised image | Signature + hash verification before execution |
| Rollback impossible | Monotonic OTP counter, version comparison |
| Deterministic execution | Fixed phase order, no dynamic dispatch |
| No information leak on failure | All failures → same FAILSAFE path, no error detail exposed |
| Zone 1 isolation after boot | MPU/MMU lockdown before EXECUTE |

---

## 8. Formal Properties

Boot state machine properties (verifiable via TLA+/nuXmv):

- **P1 (Safety):** No image with invalid signature ever reaches EXECUTE
- **P2 (Safety):** Version counter only increases, never decreases
- **P3 (Liveness):** If a valid image exists, boot eventually reaches EXECUTE
- **P4 (Safety):** Zone 2 cannot access Zone 1 memory after LOCK_BOOT

---

## 9. References

| Document | Reference |
|----------|-----------|
| Boot Flow Pseudocode | `Boot_Flow_Pseudocode.md` |
| Boot State Detail | `Boot_State_Detail.md` |
| Boot Interface | `Boot_Interface.md` |
| Boot Failure Model | `Boot_Failure_Model.md` |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` |
| Trust Model | `../../00_Architecture/Trust_Model.md` |
| Architecture Decision Record | `../../00_Architecture/Architecture_Decision_Record.md` (ADR-001) |
