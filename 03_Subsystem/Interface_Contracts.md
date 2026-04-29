# SBOP Interface Contract Specification

**Document ID:** SUB-IFC-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines formal interface contracts for all SBOP component interfaces. Each contract specifies the function signature with full type annotations, preconditions, postconditions, error conditions, and side effects.

All contracts are written in specification-level pseudocode. Concrete implementation in C, Rust, or Zig must satisfy these contracts.

---

## 2. Contract Format

Each interface uses the following specification template:

```
API_Name(param1: Type1, param2: Type2) -> Result<SuccessType, ErrorType>

    Input:  param_name : type  |  constraint
    Output: success_type       |  postcondition
    Error:  error_id           |  description

    Pre:
        condition_1
        condition_2

    Post:
        condition_1
        condition_2

    Side effects:
        description of state changes
```

---

## 3. Boot Subsystem Contracts

### 3.1 boot_verify_image

```
boot_verify_image(slot: SlotID) -> Result<ImageInfo, BootError>

    Input:  slot : SlotID  |  slot in {SLOT_A, SLOT_B}
    Output: ImageInfo      |  image is verified and ready to execute
    Error:  ERR-BOOT-CRYPTO-001    |  Signature verification failed
            ERR-BOOT-CRYPTO-002    |  Hash mismatch
            ERR-BOOT-PARSE-001     |  Invalid image header
            ERR-BOOT-VERSION-001   |  Rollback detected

    Pre:
        system_state in {RESET, INIT, SELECT_SLOT}
        slot.status in {PENDING, ACTIVE, FALLBACK}

    Post:
        if Ok(image_info):
            image_info.status == ACTIVE
            image_info.firmware_version >= previous_version
        if Err:
            system_state == FAILSAFE

    Side effects:
        - Updates stored version counter if new version is higher
        - Sets slot status to ACTIVE on success
        - Enters FAILSAFE on error (irreversible)
        - Logs boot metrics (success or failure)
```

### 3.2 boot_get_image_info

```
boot_get_image_info(slot: SlotID) -> Result<ImageInfo, BootError>

    Input:  slot : SlotID  |  slot in {SLOT_A, SLOT_B}
    Output: ImageInfo      |  metadata for firmware in slot
    Error:  ERR-STOR-READ-001  |  Cannot read slot metadata

    Pre:
        slot != SLOT_INVALID

    Post:
        Ok(ImageInfo) with all fields populated from slot metadata
        slot contents unchanged

    Side effects:
        none
```

### 3.3 boot_set_pending_image

```
boot_set_pending_image(slot: SlotID) -> Result<(), BootError>

    Input:  slot : SlotID  |  slot in {SLOT_A, SLOT_B}
    Output: ()             |  pending image set
    Error:  ERR-STOR-WRITE-001 |  Cannot write boot slot selection

    Pre:
        slot.status == PENDING
        slot != currently_active_slot()

    Post:
        boot_slot_selection == slot
        On next reset, boot will verify and execute this slot

    Side effects:
        - Writes boot slot selection to persistent storage
        - Previous active slot becomes FALLBACK
```

### 3.4 boot_enter_failsafe

```
boot_enter_failsafe(reason: BootError) -> Never

    Input:  reason : BootError  |  error that triggered fail-safe
    Output: Never               |  does not return

    Pre:
        reason.severity in {FATAL, ERROR}

    Post:
        system_state == FAILSAFE (permanent)
        No firmware executed

    Side effects:
        - Logs error
        - Zeroizes boot workspace
        - Signals fail-safe to external monitor
        - May disable debug port (if tamper-related)
```

### 3.5 boot_lock

```
boot_lock() -> Result<(), BootError>

    Input:  none
    Output: ()             |  boot locked
    Error:  ERR-HW-FUSE-001    |  Memory protection enable failed

    Pre:
        system_state in {MARK_ACTIVE}
        image_successfully_verified()

    Post:
        Zone 1 memory is read-only
        Boot configuration is immutable until next reset
        No further boot-time modifications possible

    Side effects:
        - Enables memory protection
        - May disable boot-mode interrupts
        - Transfers control to EXECUTE phase
```

---

## 4. Crypto Subsystem Contracts

### 4.1 crypto_verify_signature

```
crypto_verify_signature(
    data: &[u8],
    sig: &[u8],
    sig_len: u16,
    sig_type: SigType,
    key: KeyRef
) -> Result<bool, CryptoError>

    Input:  data : &[u8]     |  data that was signed
            sig : &[u8]      |  signature data
            sig_len : u16    |  length of sig in bytes
            sig_type : SigType | algorithm type
            key : KeyRef      |  key to verify with; usage == KEY_USAGE_VERIFY
    Output: bool             |  true = signature valid, false = invalid
    Error:  ERR-CRYPTO-SIG-001   |  Invalid signature format
            ERR-CRYPTO-SIG-002   |  Signature too large
            ERR-CRYPTO-SIG-003   |  Invalid key format

    Pre:
        data.len > 0
        sig.len == sig_len
        sig_len > 0
        key.key_type == IMAGE_VERIFY
        key.key_usage includes KEY_USAGE_VERIFY

    Post:
        Ok(true) : signature cryptographically verified
        Ok(false): signature does not verify (not an error — contract-level result)
        Err: unrecoverable format or key error

    Timing Contract:
        Execution time MUST be constant regardless of:
            - Whether signature is valid or invalid
            - Data content
            - Key value
        Max time: 20 ms for Ed25519 verify at 400 MHz

    Side effects:
        none
```

### 4.2 crypto_compute_hash

```
crypto_compute_hash(
    data: &[u8],
    algo: HashAlgo
) -> Result<[u8; 32], CryptoError>

    Input:  data : &[u8]     |  data to hash
            algo : HashAlgo  |  SHA256 only
    Output: [u8; 32]         |  32-byte hash digest
    Error:  ERR-CRYPTO-HASH-002  |  Unsupported algorithm

    Pre:
        data.len > 0
        algo == SHA256

    Post:
        output == SHA-256(data)
        output is deterministic (same input → same output)

    Timing Contract:
        Execution time MUST be constant for fixed input length
        Max time: 100 ms for 1 MB payload at 100 MHz

    Side effects:
        none
```

### 4.3 crypto_constant_time_compare

```
crypto_constant_time_compare(
    a: &[u8],
    b: &[u8],
    len: usize
) -> bool

    Input:  a, b : &[u8]  |  buffers to compare
            len : usize     |  number of bytes to compare
    Output: bool            |  true if all bytes equal

    Pre:
        a.len >= len
        b.len >= len
        len > 0

    Post:
        output == (a[0..len] == b[0..len])

    Timing Contract:
        Execution cycle count MUST depend only on len (public), NOT on buffer content
        Every byte position must be compared regardless of earlier differences
        No early return on mismatch

    Side effects:
        none
```

### 4.4 crypto_derive_key

```
crypto_derive_key(
    input_key_material: &[u8],
    salt: &[u8],
    info: &[u8],
    output_len: u16
) -> Result<KeyMaterial, CryptoError>

    Input:  input_key_material : &[u8]  |  source key (KD or KR)
            salt : &[u8]                |  KDF salt
            info : &[u8]                |  KDF info/context string
            output_len : u16            |  desired output key length
    Output: KeyMaterial                 |  derived key
    Error:  ERR-CRYPTO-KDF-001          |  KDF internal error
            ERR-CRYPTO-KDF-002          |  Insufficient input entropy
            ERR-CRYPTO-KDF-003          |  Invalid salt

    Pre:
        input_key_material.len >= 16
        salt.len > 0
        output_len >= 32

    Post:
        output_material is HKDF-SHA-256(input_key_material, salt, info, output_len)
        Deterministic: same inputs → same output

    Timing Contract:
        Constant-time with respect to input_key_material and output_len

    Side effects:
        none
```

---

## 5. OTA Subsystem Contracts

### 5.1 ota_authenticate

```
ota_authenticate(
    device_id: DeviceID,
    challenge: &[u8; 32]
) -> Result<AuthToken, OTAError>

    Input:  device_id : DeviceID  |  device identity
            challenge : &[u8; 32] |  fresh challenge from backend
    Output: AuthToken             |  authentication token
    Error:  ERR-OTA-AUTH-001      |  Backend rejected authentication
            ERR-OTA-AUTH-003      |  Update not authorized

    Pre:
        device_id is valid and provisioned
        challenge is fresh (nonce)

    Post:
        if Ok(token):
            token.expires_at > current_time
            token.data valid for subsequent API calls
            Update negotiation authorized for duration of token
        if Err:
            No token issued; retry possible after backoff

    Side effects:
        - Increments authentication attempt counter
        - Logs attempt to backend
```

### 5.2 ota_download_image

```
ota_download_image(
    url: &UrlString,
    expected_hash: &[u8; 32],
    slot: SlotID
) -> Result<(), OTAError>

    Input:  url : &UrlString         |  download location
            expected_hash : &[u8;32] |  expected SHA-256 of image
            slot : SlotID            |  target (must be inactive)
    Output: ()                       |  download complete and verified
    Error:  ERR-OTA-DOWNLOAD-001     |  Download interrupted
            ERR-OTA-DOWNLOAD-002     |  Hash mismatch
            ERR-OTA-DOWNLOAD-003     |  Insufficient storage

    Pre:
        slot != currently_active_slot()
        slot.status in {EMPTY, INVALID}
        internet_connectivity_available()

    Post:
        if Ok:
            slot.status == VERIFIED
            slot contains downloaded image
            SHA-256(image) == expected_hash
        if Err (hash mismatch):
            slot.status == INVALID
            Downloaded data discarded

    Side effects:
        - Writes firmware image to storage
        - May trigger storage sector erase
        - Updates slot status

    Timing:
        Duration depends on image size and bandwidth
        Must support resume from interrupted download (HTTP Range)
```

### 5.3 ota_activate_pending

```
ota_activate_pending() -> Result<(), OTAError>

    Input:  none
    Output: ()             |  pending firmware set as boot target
    Error:  ERR-OTA-INSTALL-001 |  Pending slot invalid

    Pre:
        exists slot with slot.status == VERIFIED
        slot firmware version > current_active_version

    Post:
        boot_slot_selection set to pending slot
        On next reset: boot will verify and activate

    Side effects:
        - Writes boot slot selection to persistent storage
        - Triggers system reset (or awaits external reset)
```

### 5.4 ota_rollback

```
ota_rollback() -> Result<(), OTAError>

    Input:  none
    Output: ()             |  system reverted to previous firmware
    Error:  ERR-OTA-ACTIVATE-001 |  No valid fallback image

    Pre:
        exists slot with slot.status == FALLBACK

    Post:
        boot_slot_selection set to fallback slot
        Previously active slot marked INVALID

    Side effects:
        - Writes boot slot selection to storage
        - Increments rollback counter
        - Reports rollback to backend
        - Triggers system reset
```

---

## 6. Identity Subsystem Contracts

### 6.1 identity_get_device_id

```
identity_get_device_id() -> Result<DeviceID, IdentityError>

    Input:  none
    Output: DeviceID        |  device unique identifier
    Error:  ERR-ID-PROV-001 |  Device not yet provisioned

    Pre:
        device_provisioning_state == REGISTERED or LOCKED

    Post:
        Ok(uid) contains the immutable device UID assigned during provisioning

    Side effects:
        none
```

### 6.2 identity_enter_provisioning

```
identity_enter_provisioning(
    station_auth: ProvisioningAuth
) -> Result<ProvisioningSession, IdentityError>

    Input:  station_auth : ProvisioningAuth  |  provisioning station credentials
    Output: ProvisioningSession              |  active provisioning session
    Error:  ERR-ID-PROV-001                  |  Already provisioned
            ERR-ID-PROV-003                  |  Station auth failed

    Pre:
        device_provisioning_state == UNINITIALIZED
        station_auth is valid

    Post:
        provisioning session established
        Device ready to receive key material

    Side effects:
        - Generates device UID (from TRNG)
        - Generates attestation key pair
        - Sets state to PROVISIONING
```

### 6.3 identity_lock

```
identity_lock(
    lock_attestation: &[u8; 64]
) -> Result<(), IdentityError>

    Input:  lock_attestation : &[u8; 64]  |  signature proving lock was authorized
    Output: ()                             |  device identity locked
    Error:  ERR-ID-PROV-001                |  Already locked

    Pre:
        device_provisioning_state == REGISTERED

    Post:
        device_provisioning_state == LOCKED
        No further identity modifications possible

    Side effects:
        - Locks identity permanently (fuse or OTP)
        - Disables debug port
        - Stores lock attestation for audit
```

---

## 7. Backend Interface Contracts (Abstract)

### 7.1 backend_register_device

```
backend_register_device(
    device_uid: DeviceID,
    attestation_proof: &[u8; 64],
    firmware_version: u32,
    firmware_hash: &[u8; 32]
) -> Result<RegistrationToken, BackendError>

    Input:  device_uid, attestation_proof, version, hash
    Output: RegistrationToken (confirms registration)
    Error:  BackendError::AlreadyRegistered, ::InvalidAttestation

    Contract:
        Registration is idempotent (calling twice with same UID is OK)
        Backend rejects duplicate UIDs with different attestation proofs
```

### 7.2 backend_request_update

```
backend_request_update(
    device_uid: DeviceID,
    current_version: u32,
    auth_token: AuthToken
) -> Result<UpdateInfo, BackendError>

    Input:  device_uid, current_version, auth_token
    Output: UpdateInfo (or None if no update available)
    Error:  BackendError::AuthFailed, ::NotAuthorized

    Contract:
        Backend verifies auth_token before serving update
        Backend enforces deployment policy (canary %, rollout %)
        Backend logs request for audit
```

---

## 8. Contract Compliance

### 8.1 Verification Rules

| Rule | Description |
| --- | --- |
| CMPL-001 | All preconditions must be checked by caller (assert on violation) |
| CMPL-002 | All postconditions must be guaranteed by callee |
| CMPL-003 | Error conditions are exhaustive (no unspecified errors) |
| CMPL-004 | Timing contracts are verified by measurement (→ `Side_Channel_Countermeasures.md`) |
| CMPL-005 | Side effects are documented and bounded |

### 8.2 Contract Versioning

Contracts follow semantic versioning:
- **Major**: Breaking changes to signatures, pre/post conditions, or error sets
- **Minor**: New optional fields, new error codes (non-breaking additions)
- **Patch**: Documentation clarifications only
