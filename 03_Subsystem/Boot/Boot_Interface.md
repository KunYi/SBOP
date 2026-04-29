# Boot Subsystem Interface Specification

**Document ID:** SUB-BOOT-IF-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines the abstract interfaces required by the Boot subsystem for cryptographic verification, storage access, identity retrieval, and system control. These interfaces are the contract between the Boot subsystem and the platform implementation.

→ Formal contracts in `../Interface_Contracts.md` §3 (boot), §4 (crypto), §6 (identity)

---

## 2. Crypto Interface

### 2.1 verify_signature

```
Function: verify_signature(data: &[u8], signature: &SignatureBlock, public_key: &KeyRef) → Result<(), BootError>

Purpose: Verify cryptographic signature over firmware image.

Preconditions:
  - data points to image payload (header + payload, excluding signature block)
  - signature.algorithm matches one of the supported algorithms (Ed25519)
  - public_key resolves to a valid verification key (KI public key)

Postconditions:
  - Ok(()) → signature is cryptographically valid for data under public_key
  - Err(BootError::SignatureInvalid) → signature verification failed
  - Err(BootError::CryptoEngineError) → hardware crypto engine fault

Timing contract:
  - Execution time MUST be independent of data, signature, and key values
  - No early return on verification failure (constant-time per Side_Channel_Countermeasures.md §3)

Errors:
  - ERR-BOOT-CRYPTO-001: Signature verification failed (cryptographic)
  - ERR-BOOT-CRYPTO-003: Unsupported signature algorithm
  - ERR-BOOT-CRYPTO-006: Crypto engine hardware fault
  - ERR-BOOT-CRYPTO-004: Key reference invalid or revoked
```

### 2.2 compute_hash

```
Function: compute_hash(data: &[u8]) → Result<[u8; 32], BootError>

Purpose: Compute SHA-256 hash of image payload.

Preconditions:
  - data points to valid memory region of image_size bytes

Postconditions:
  - Ok(hash) → hash = SHA-256(data)
  - Err(BootError::CryptoEngineError) → hardware fault

Errors:
  - ERR-BOOT-CRYPTO-006: Crypto engine hardware fault
```

---

## 3. Storage Interface

### 3.1 read_image

```
Function: read_image(slot: SlotID) → Result<Vec<u8>, BootError>

Purpose: Read firmware image from specified slot into RAM for verification.

Preconditions:
  - slot is SlotA or SlotB
  - slot flash region is accessible (MPU not yet configured)

Postconditions:
  - Ok(data) → complete image bytes from slot
  - Err(BootError::SlotEmpty) → slot has no image (EMPTY status)
  - Err(BootError::SlotInvalid) → slot marked INVALID

Errors:
  - ERR-STOR-READ-001: Slot read failed (hardware)
  - ERR-BOOT-PARSE-004: Image exceeds maximum size
  - ERR-STOR-READ-001: Slot metadata CRC mismatch (→ Error_Code_Catalog.md)
```

### 3.2 read_slot_metadata

```
Function: read_slot_metadata(slot: SlotID) → Result<SlotMetadata, BootError>

Purpose: Read slot metadata to determine slot status, version, and integrity.

Errors:
  - ERR-STOR-READ-001: Metadata read failed
  - ERR-STOR-READ-001: Metadata CRC mismatch (→ Error_Code_Catalog.md)
```

### 3.3 mark_slot_active

```
Function: mark_slot_active(slot: SlotID) → Result<(), BootError>

Purpose: Atomic write of ACTIVE status to the specified slot after successful verification.

Preconditions:
  - Boot state machine is in MARK_ACTIVE phase
  - Image in slot has passed all verification steps

Postconditions:
  - Ok(()) → slot.status = ACTIVE; other slot status ≠ ACTIVE
  - Operation is atomic (must survive power loss mid-write)

Errors:
  - ERR-STOR-WRITE-001: Slot status write failed
```

### 3.4 mark_slot_invalid

```
Function: mark_slot_invalid(slot: SlotID) → Result<(), BootError>

Purpose: Mark a slot as INVALID after verification failure.

Postconditions:
  - slot.status = INVALID
  - Device transitions to FAILSAFE if no valid slot remains
```

---

## 4. Identity Interface

### 4.1 get_device_identity

```
Function: get_device_identity() → Result<DeviceID, BootError>

Purpose: Retrieve the device's unique cryptographic identity.

Postconditions:
  - Ok(id) → id.uid is the provisioned 128-bit UID
  - Ok(id) → id.state is LOCKED (provisioning complete)
  - Err(BootError::IdentityNotProvisioned) → device not yet provisioned
```

### 4.2 get_root_trust

```
Function: get_root_trust() → Result<KeyRef, BootError>

Purpose: Get a reference to the root verification key (KI public key). The returned KeyRef is a handle — raw key material is never exposed.

Postconditions:
  - Ok(key_ref) → key_ref can be passed to verify_signature
  - Key material remains in secure storage (key handle only)
```

---

## 5. Control Interface

### 5.1 jump_to_firmware

```
Function: jump_to_firmware(entry_point: u32, stack_pointer: u32) → !

Purpose: Transfer execution to verified application firmware. This function does not return.

Preconditions:
  - Boot state machine is in EXECUTE phase
  - All verification steps passed
  - MPU/MMU configured for Zone 1/2 isolation

Postconditions:
  - Application firmware executing at entry_point
  - CPU registers cleared (except SP and PC)
  - Zone 1 memory locked read-only
  - Boot-mode peripherals disabled

Safety:
  - This is the security boundary crossing — must clear all local state before jump
```

### 5.2 enter_failsafe

```
Function: enter_failsafe(error: BootError) → !

Purpose: Enter FAILSAFE state. This function does not return.

Preconditions:
  - A boot verification step has failed
  - Error code indicates the specific failure

Postconditions:
  - Device in FAILSAFE state
  - No application code executing
  - Tamper log updated with error
  - Recovery mechanism active (if available)
  - On next power cycle, boot attempts recovery or remains in FAILSAFE
```

### 5.3 enter_tamper_lock

```
Function: enter_tamper_lock(tamper_type: TamperEventType) → !

Purpose: Enter permanent TAMPER_LOCK state after critical tamper detection. Keys are zeroized.

Postconditions:
  - All keys in secure element zeroized
  - Device permanently inoperative
  - TAMPER_LOCK state is irreversible
```

---

## 6. Tamper Interface

### 6.1 sample_tamper_sensors

```
Function: sample_tamper_sensors() → Result<TamperStatus, BootError>

Purpose: Read all tamper detection sensors (enclosure, voltage, clock, temperature, glitch counters).

Postconditions:
  - Ok(status) → status indicates which sensors (if any) detected tampering
  - Glitch counter values included in status
```

### 6.2 handle_tamper

```
Function: handle_tamper(status: TamperStatus) → Result<(), BootError>

Purpose: Process tamper sensor readings and execute appropriate response.

Response levels:
  - TAMPER_NONE: No action
  - TAMPER_MINOR: Log event, increment tamper counter
  - TAMPER_MAJOR: Log event, enter FAILSAFE
  - TAMPER_CRITICAL: Log event, zeroize keys, enter TAMPER_LOCK
```

---

## 7. Constraints

| ID | Constraint |
| --- | --- |
| C-BOOT-01 | All interfaces must be deterministic: same input → same output |
| C-BOOT-02 | No interface may expose raw key material (KeyRef handles only) |
| C-BOOT-03 | No interface may allow bypass of verification sequence |
| C-BOOT-04 | jump_to_firmware must clear all CPU registers except SP and PC |
| C-BOOT-05 | enter_failsafe and enter_tamper_lock must be non-returnable |
| C-BOOT-06 | All crypto operations must be constant-time |
| C-BOOT-07 | Boot must complete within timing budget (see Boot_Flow_Pseudocode.md §11) |
| C-BOOT-08 | No dynamic memory allocation after INIT phase |

---

## 8. References

| Document | Reference |
| --- | --- |
| Formal Interface Contracts | `../Interface_Contracts.md` §3-4 |
| Boot Flow Pseudocode | `Boot_Flow_Pseudocode.md` |
| Data Structures | `../../02_System_Design/Data_Structures.md` §3-6 |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §4 |
| Side-Channel Countermeasures | `../../04_Security/Side_Channel_Countermeasures.md` |
