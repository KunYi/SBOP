# Identity Interface Specification

**Document ID:** SUB-ID-IF-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the abstract interfaces for device identity operations: identity retrieval, key access, provisioning control, and backend registration.

→ Formal contracts in `../Interface_Contracts.md` §6 (identity), §8 (provisioning)

---

## 2. Device Identity Interface

### 2.1 get_device_identity

```
Function: get_device_identity() → Result<DeviceID, IdentityError>

Purpose: Return the device's provisioned cryptographic identity.

Preconditions:
  - Device has been provisioned (state ≥ PROVISIONING)

Postconditions:
  - Ok(id) where id.state == LOCKED → identity finalized, immutable
  - Ok(id) where id.state == PROVISIONING → identity in progress (factory only)
  - Err(IdentityError::NotProvisioned) → device never provisioned

Data returned:
  - uid: [u8; 16] — unique device identifier (TRNG-derived)
  - state: IdentityState — current provisioning state
  - kr_commitment: [u8; 32] — HMAC(KR, "COMMIT") enabling KR verification

Errors:
  - ERR-ID-IDENT-001: Identity not provisioned
  - ERR-ID-IDENT-002: Identity storage corrupted
```

### 2.2 get_device_uid_hash

```
Function: get_device_uid_hash() → Result<[u8; 16], IdentityError>

Purpose: Return HMAC(KD, UID) for anonymized telemetry and backend identification.

Rationale: Telemetry uses hashed identity to prevent UID exposure while enabling
backend deduplication and fleet tracking. See Privacy constraint C-ID-05.
```

---

## 3. Key Access Interface

### 3.1 get_root_trust

```
Function: get_root_trust() → Result<KeyRef, IdentityError>

Purpose: Get a reference handle to the root verification key (KI public key).

Postconditions:
  - Ok(key_ref) → handle to KI public key for signature verification
  - Raw key material is never exposed (KeyRef is an opaque handle)

Errors:
  - ERR-ID-KEY-001: Root trust not established
  - ERR-ID-KEY-002: Key storage access denied (MPU violation)
```

### 3.2 derive_device_key

```
Function: derive_device_key(context: KeyContext) → Result<KeyRef, IdentityError>

Purpose: Derive a sub-key from KD for a specific purpose.

Preconditions:
  - Device is provisioned (KD exists)
  - context specifies a valid key type (AUTH, DEBUG, STORAGE)

Postconditions:
  - Ok(key_ref) → derived key handle for requested context
  - Key derived via HKDF(KD, context_info) per Key_Derivation.md

Valid contexts:
  - KeyContext::Auth → KD_Auth (OTA mutual TLS)
  - KeyContext::Debug → KD_Debug (debug challenge-response)
  - KeyContext::Storage → KD_Storage (optional flash encryption)

Errors:
  - ERR-ID-KEY-003: KD not available (not provisioned or zeroized)
  - ERR-ID-KEY-004: Invalid key context
  - ERR-ID-KEY-005: Key derivation failed (crypto engine error)
```

### 3.3 verify_key_presence

```
Function: verify_key_presence() → Result<(), IdentityError>

Purpose: Verify that KD is present and accessible (challenge-response with secure element).

Used during boot to confirm key material has not been zeroized by tamper response.
```

---

## 4. Provisioning Control Interface

### 4.1 enter_provisioning_mode

```
Function: enter_provisioning_mode() → Result<(), IdentityError>

Purpose: Transition device from UNINITIALIZED to PROVISIONING state.

Preconditions:
  - Device state is UNINITIALIZED
  - Called from provisioning firmware

Postconditions:
  - Ok(()) → device state = PROVISIONING
  - UID generated from TRNG
  - Device ready for key injection

Errors:
  - ERR-ID-PROV-001: Device already provisioned
  - ERR-ID-PROV-002: TRNG failure (entropy source unhealthy)
```

### 4.2 inject_device_key

```
Function: inject_device_key(kd: &KeyMaterial, kd_debug: &KeyMaterial, kd_auth: &KeyMaterial) → Result<(), IdentityError>

Purpose: Inject device keys into secure element during provisioning.

Preconditions:
  - Device state is PROVISIONING
  - Channel is encrypted (provisioning station to device)
  - Keys derived from KR + UID (verified by station)

Postconditions:
  - Ok(()) → KD, KD_Debug, KD_Auth stored in secure element
  - Keys accessible only via KeyRef handles

Errors:
  - ERR-ID-PROV-003: Key injection failed (secure element error)
  - ERR-ID-PROV-004: Key verification failed (challenge-response mismatch)
```

### 4.3 lock_device_identity

```
Function: lock_device_identity() → Result<(), IdentityError>

Purpose: Finalize provisioning — lock identity, disable debug, transition to operational state.

Preconditions:
  - Device state is PROVISIONING
  - All keys injected and verified
  - Self-tests passed

Postconditions:
  - Ok(()) → device state = LOCKED (irreversible)
  - Identity cannot be modified
  - Debug port disabled (if configured)

Errors:
  - ERR-ID-PROV-005: Lock failed (hardware error)
  - ERR-ID-PROV-006: Self-test failed before lock

---

## 5. Identity State Query Interface

### 5.1 get_identity_state

```
Function: get_identity_state() → Result<IdentityState, IdentityError>

Purpose: Return the current identity lifecycle state.

Possible states:
  - IdentityState::Uninitialized — factory-fresh, no identity
  - IdentityState::Provisioning — identity being created
  - IdentityState::Registered — backend registration complete
  - IdentityState::Locked — identity immutable, operational
  - IdentityState::Revoked — backend-revoked, all operations denied
  - IdentityState::TamperLocked — keys zeroized, permanently inoperative
```

### 5.2 revoke_identity

```
Function: revoke_identity(authorization: RevokeToken) → Result<(), IdentityError>

Purpose: Backend-initiated revocation of device identity.

Preconditions:
  - authorization.token verifies with backend public key
  - authorization.target_uid matches device UID

Postconditions:
  - Ok(()) → device state = REVOKED
  - All operations denied (no boot, no OTA, no debug)

Errors:
  - ERR-ID-AUTH-001: Revocation authorization invalid
```
```

---

## 5. Backend Registration Interface

### 5.1 send_identity_proof

```
Function: send_identity_proof() → Result<RegistrationResponse, IdentityError>

Purpose: Generate and send cryptographic proof of identity to backend for registration.

Proof generation:
  attestation = Sign(KD, UID || timestamp || backend_nonce)

Postconditions:
  - Ok(response) → backend accepted registration
  - Ok(response) where response.status == DUPLICATE → UID collision detected
```

### 5.2 receive_registration_status

```
Function: receive_registration_status() → Result<RegistrationResponse, IdentityError>

Purpose: Poll or receive callback for registration status after sending proof.
```

---

## 6. Anti-Cloning Interface

### 6.1 verify_unique_identity

```
Function: verify_unique_identity() → Result<(), IdentityError>

Purpose: Run local anti-cloning checks:
  1. UID entropy meets minimum (NIST SP 800-90B health test passed)
  2. KD is accessible and responds to challenge
  3. Identity state is LOCKED (not still in provisioning)
  4. No duplicate UID detected (requires backend check)

Errors:
  - ERR-ID-CLONE-001: UID entropy insufficient (possible weak TRNG)
  - ERR-ID-CLONE-002: Backend reports duplicate UID
```

---

## 7. Constraints

| ID | Constraint |
| --- | --- |
| C-ID-01 | Identity must be immutable after lock_device_identity (no modification, no re-provisioning) |
| C-ID-02 | Raw key material (KD, KR) must never be exposed outside KeyRef opaque handles |
| C-ID-03 | UID must have ≥ 128 bits of entropy (TRNG + NIST SP 800-90B health tests) |
| C-ID-04 | All identity interfaces must be deterministic |
| C-ID-05 | Telemetry must use anonymized identity (UID hash, not UID itself) |
| C-ID-06 | Debug unlock must not expose KD or derived keys (even with full debug access) |
| C-ID-07 | Provisioning mode must be detectable (device must report PROVISIONING state) |

---

## 8. Lifecycle States

```
UNINITIALIZED → PROVISIONING → REGISTERED → LOCKED
                                       ↓
                                  TAMPER_LOCKED
                                  (keys zeroized, permanent)
```

→ See `Provisioning_Flow.md` §7 for state machine details and transitions.

---

## 9. References

| Document | Reference |
| --- | --- |
| Formal Interface Contracts | `../Interface_Contracts.md` §6, §8 |
| Provisioning Flow | `Provisioning_Flow.md` |
| Identity Overview | `Identity_Overview.md` |
| Anti-Cloning | `Anti_Cloning.md` |
| Key Derivation | `../Crypto/Key_Derivation.md` |
| Data Structures | `../../02_System_Design/Data_Structures.md` §DeviceID, KeyRef |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §7 |
| Manufacturing Security | `../../04_Security/Manufacturing_Security.md` |
