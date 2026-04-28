# Secure Debug Architecture

**Document ID:** SEC-DBG-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the architectural requirements for secure debug access in SBOP. It specifies the debug port lifecycle, authentication protocol, access levels, and the interface between SBOP and the platform debug hardware.

---

## 2. Debug Access Threat Model

### 2.1 Threat Scenarios

| Threat ID | Scenario | Impact |
| --- | --- | --- |
| DBG-T-001 | Attacker connects to unlocked JTAG/SWD after provisioning | Full device access: memory readout, code injection, key extraction |
| DBG-T-002 | Attacker exploits debug port before permanent lock | Race condition during manufacturing/factory test phase |
| DBG-T-003 | Attacker uses debug to inject breakpoint during verification | Bypass signature check; force EXECUTE path |
| DBG-T-004 | Attacker uses debug auth credentials from compromised provisioning station | Unauthorized debug unlock in field |
| DBG-T-005 | Attacker performs fault injection during debug unlock sequence | Bypass debug authentication |

### 2.2 Attack Surface

The debug port is the highest-value attack surface on the device:

- **JTAG** (IEEE 1149.1): Full scan chain access, memory access, CPU control
- **SWD** (ARM Serial Wire Debug): ARM equivalent of JTAG, similar capabilities
- **cJTAG** (IEEE 1149.7): Compact JTAG, same security concerns
- **UART bootloader**: Often provides debug/update access with limited authentication
- **USB DFU**: Device Firmware Upgrade mode with potential debug access

---

## 3. Debug Port Lifecycle

### 3.1 Lifecycle States

```
                    ┌──────────┐
        Power-On ──>│  OPEN    │  Development / prototyping
                    └────┬─────┘
                         │ Security fuse not blown
                    ┌────▼─────┐
                    │  AUTH    │  Manufacturing test / debug
                    └────┬─────┘
                         │ Challenge-response auth enabled
                    ┌────▼─────┐
    Provisioning ──>│  LOCKED  │  Production device
                    └────┬─────┘
                         │ Permanent lock + security fuses
                    ┌────▼─────┐
                    │  DEAD    │  Fully disabled (no recovery)
                    └──────────┘
```

### 3.2 Lifecycle Phase Details

#### Phase 1: OPEN (Development)

| Attribute | Value |
| --- | --- |
| State | Debug port fully accessible |
| Authentication | None |
| Access Level | Full (memory, registers, CPU control) |
| Transition Trigger | Security fuse programmed or debug lock enabled |
| Next State | AUTH or LOCKED |

#### Phase 2: AUTH (Manufacturing Test)

| Attribute | Value |
| --- | --- |
| State | Debug port requires authentication |
| Authentication | Challenge-response using device-specific key (KD) |
| Access Level | Configurable (Level 0-2 based on auth result) |
| Transition Trigger | Provisioning completed, device registered |
| Next State | LOCKED |

#### Phase 3: LOCKED (Production)

| Attribute | Value |
| --- | --- |
| State | Debug port permanently disabled for normal operation |
| Authentication | Authorized unlock requires device-specific credentials + backend authorization |
| Access Level | Level 0 (fully locked) by default |
| Back Door | Time-limited authorized unlock for failure analysis (see Section 5) |
| Next State | DEAD (if permanent disable performed) |

#### Phase 4: DEAD (End of Life / Compromised)

| Attribute | Value |
| --- | --- |
| State | Debug port irreversibly disabled |
| Authentication | None possible |
| Recovery | None — physical replacement only |

---

## 4. Debug Authentication Protocol

### 4.1 Challenge-Response Protocol

```
Device                                    Debug Host
  │                                           │
  │  ← Debug Connect Request                  │
  │                                           │
  │  → Challenge (32-byte random nonce)       │
  │     + Device UID                          │
  │                                           │
  │  ← Response = MAC(KD, Challenge || UID)   │
  │     + Requested Access Level              │
  │                                           │
  │  Verify Response locally                  │
  │  Grant access at requested level          │
  │  → Auth Result (Success / Failure)        │
  │                                           │
```

### 4.2 Protocol Requirements

| Requirement | Description |
| --- | --- |
| DBG-AUTH-001 | Challenge must be fresh 32-byte random value (from TRNG) |
| DBG-AUTH-002 | Response must be computed as HMAC-SHA-256(KD, challenge || UID || access_level) |
| DBG-AUTH-003 | KD used for debug auth must be different from operational KD (debug-specific derived key) |
| DBG-AUTH-004 | Maximum 3 authentication attempts before rate-limit lockout (exponential backoff) |
| DBG-AUTH-005 | After 10 failed attempts: permanent lock (DEAD state for debug) |

### 4.3 Key Derivation for Debug

```
KD_Debug = HKDF-Extract(
    salt = "SBOP_DEBUG_AUTH_v1",
    ikm = KD  // Device Key
)

KD_Debug is used ONLY for debug authentication, never for firmware verification.
```

---

## 5. Debug Access Levels

### 5.1 Access Level Definitions

| Level | Name | Allowed Operations | Typical Use Case |
| --- | --- | --- | --- |
| **Level 0** | Fully Locked | No debug access | Production device |
| **Level 1** | Memory Read-Only | Read memory regions (non-security); no CPU control | Failure analysis (read logs/state) |
| **Level 2** | Full Debug | All debug operations | Manufacturing test and development |
| **Level 3** | Permanent Unlock | All debug; irreversible | Device decommissioning |

### 5.2 Level Access Matrix

| Operation | Level 0 | Level 1 | Level 2 | Level 3 |
| --- | --- | --- | --- | --- |
| Connect to debug port | No | Yes (auth) | Yes (auth) | Yes (permanent) |
| Read application memory | No | Yes (non-secure only) | Yes | Yes |
| Read secure memory (Zone 1) | No | No | No | No |
| Read key material | No | No | No | No |
| Set breakpoints | No | No | Yes | Yes |
| Single-step CPU | No | No | Yes | Yes |
| Modify memory | No | No | Yes | Yes |
| Modify fuses/OTP | No | No | No | Yes |
| Access via JTAG/SWD | No | Yes (auth) | Yes (auth) | Yes |

### 5.3 Key Material Is Always Protected

Regardless of debug access level, SBOP's key material (KR, KD, signing keys) must never be accessible via debug. This is enforced by:

- Key storage in hardware-protected region (secure element, TEE, or protected flash)
- Debug access to key storage region is blocked at hardware level
- Key registers are cleared before debug control is granted

---

## 6. Authorized Debug Unlock (Field Failure Analysis)

### 6.1 Unlock Procedure

```
1. Field technician identifies device requiring debug analysis
2. Technician requests debug unlock from backend with justification
3. Backend verifies: device identity, technician authorization, business justification
4. Backend issues time-limited unlock token:
   unlock_token = {
       device_uid,
       technician_id,
       access_level (Level 1 or 2),
       expiry_timestamp,
       backend_signature
   }
5. Technician presents unlock token to device via debug host
6. Device verifies backend_signature using backend public key
7. Device verifies device_uid matches, expiry not reached
8. Device transitions debug state: LOCKED → Level 1 or 2 (time-limited)
9. Debug access automatically relocks at expiry_timestamp or next reset
```

### 6.2 Unlock Security Requirements

| Requirement | Description |
| --- | --- |
| DBG-UL-001 | Unlock token must be signed by backend (ECDSA or Ed25519) |
| DBG-UL-002 | Token must be single-use (device records used token IDs) |
| DBG-UL-003 | Token expiry must not exceed 24 hours from issuance |
| DBG-UL-004 | Debug relock must occur on any of: token expiry, device reset, power cycle |
| DBG-UL-005 | Unlock event must be logged in device audit record |
| DBG-UL-006 | Maximum 5 authorized unlocks per device lifetime |

---

## 7. Debug Event Logging

| Event | Log Data | Storage |
| --- | --- | --- |
| Debug connect attempt (locked) | Timestamp, source | Tamper event log |
| Debug auth success | Timestamp, access level, token ID | Audit log |
| Debug auth failure | Timestamp, attempt count | Tamper event log |
| Debug relock | Timestamp, reason | Audit log |
| Authorized unlock request | Technician ID, justification, token ID | Backend + device |

---

## 8. Platform Integration

### 8.1 Interface Requirements

```
debug_set_state(state: DebugState) → Result<(), HW_Error>
debug_get_state() → DebugState
debug_auth(challenge: &[u8; 32]) -> Result<AuthResponse, DebugAuthError>
debug_lock_permanent() → Never  // Irreversible; returns only on failure
```

### 8.2 Hardware Requirements

| Requirement | Description |
| --- | --- |
| DBG-HW-001 | Platform must support disabling debug port via fuse or OTP bit |
| DBG-HW-002 | Debug disable mechanism must be irreversible (cannot be re-enabled by firmware) |
| DBG-HW-003 | Debug port lock state must survive power cycles and resets |
| DBG-HW-004 | Platform should support access-level gating (different debug domains for Level 1/2) |

---

## 9. Requirements Trace

| Debug Threat | SBOP Requirement | Mechanism | Verification |
| --- | --- | --- | --- |
| DBG-T-001 (Unauthorized access) | REQ-SR-DBG-001 | Debug locked after provisioning | Functional test |
| DBG-T-002 (Race condition) | REQ-SR-DBG-001 | Tamper-evident lifecycle; auth before provisioning | Timing analysis |
| DBG-T-003 (Breakpoint injection) | REQ-SR-DBG-001 | Debug disabled during boot (if locked) | Fault injection test |
| DBG-T-004 (Credential compromise) | REQ-SR-DBG-002 | Device-specific KD_Debug derivation | Key isolation analysis |
| DBG-T-005 (Auth glitch) | REQ-SR-DBG-002 | Redundant auth check; fail-safe on auth error | Fault injection test |
