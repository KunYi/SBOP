# Provisioning Flow Specification

**Document ID:** SUB-ID-PF-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines how a device transitions from untrusted to trusted.

---

## 2. Provisioning Modes

* Factory Provisioning
* Field Provisioning (optional)

---

## 3. Factory Provisioning Flow

1. Device enters provisioning mode
2. Device identity is established (UID)
3. Station derives KD = HKDF(KR, UID) via HSM; KR never leaves HSM
4. KD is injected into device secure element via encrypted channel
5. Device registers with backend
6. Device is locked into operational state

---

## 4. Backend Registration

* Device sends identity proof
* Backend verifies trust origin
* Backend binds device to account / system

---

## 5. Post-Provisioning State

* Device cannot re-enter provisioning
* Identity must be immutable

---

## 6. Failure Handling

* Provisioning failure → device remains untrusted
* Partial provisioning → must restart process

---

## 7. Provisioning Pseudocode

```
function factory_provision(provisioning_data: ProvisioningConfig) -> Result<(), ProvisioningError>:
    // ── Phase 1: Enter Provisioning Mode ──
    state = PROV_UNINITIALIZED
    
    // Verify device is in uninitialized state
    let current_state = identity_get_provisioning_state()
    assert(current_state == UNINITIALIZED, ERR-ID-PROV-001)
    
    // Authenticate provisioning station
    let station_auth = verify_provisioning_station(provisioning_data.station_cred)
    assert(station_auth.is_valid(), ERR-ID-PROV-003)
    
    state = PROV_PROVISIONING
    
    // ── Phase 2: Generate Device Identity ──
    let device_uid = generate_device_uid()
    // UUIDv4 format: 128-bit random from TRNG
    assert(device_uid.entropy_ok(), ERR-CRYPTO-RNG-001)
    
    // Generate attestation key pair
    let attestation_keypair = crypto_generate_keypair(Ed25519)
    assert(attestation_keypair.is_ok(), ERR-CRYPTO-RNG-001)
    
    // ── Phase 3: Receive Device Key from Provisioning Station ──
    // KD = HKDF(KR, UID) was derived at the station using HSM.
    // KR never leaves the HSM. Only KD is injected to the device.
    let kd_wrapped = provisioning_data.device_key_wrapped
    let kd = key_unwrap(kd_wrapped, device_transport_key)
    assert(kd.is_valid(), ERR-ID-PROV-002)
    
    // Store KD in secure element (never exposed again)
    key_store_secure(KEY_SLOT_DEVICE, kd)
    
    // ── Phase 4: Derive Sub-Keys from KD ──
    let kd_auth = crypto_derive_key(
        input_key_material = kd,
        salt = [],
        info = "SBOP-AUTH-v1",
        output_len = 32
    )
    let kd_debug = crypto_derive_key(
        input_key_material = kd,
        salt = [],
        info = "SBOP-DEBUG-v1",
        output_len = 32
    )
    assert(kd_auth.is_ok() && kd_debug.is_ok(), ERR-CRYPTO-KDF-001)
    
    key_store_secure(KEY_SLOT_AUTH, kd_auth.unwrap())
    key_store_secure(KEY_SLOT_DEBUG, kd_debug.unwrap())
    
    // ── Phase 5: Generate Attestation Proof ──
    let attestation_data = concat(device_uid.bytes(), [
        provisioning_data.firmware_version,
        provisioning_data.firmware_hash,
        provisioning_data.timestamp
    ])
    
    let attestation_proof = crypto_sign(
        data = attestation_data,
        key = attestation_keypair.private_key
    )
    
    // ── Phase 6: Register with Backend ──
    let registration_result = backend_register_device(
        device_uid = device_uid,
        attestation_proof = attestation_proof,
        attestation_pubkey = attestation_keypair.public_key,
        firmware_version = provisioning_data.firmware_version,
        firmware_hash = provisioning_data.firmware_hash,
        station_id = provisioning_data.station_id,
        operator_id = provisioning_data.operator_id
    )
    assert(registration_result.is_ok(), ERR-ID-PROV-002)
    
    let registration_token = registration_result.unwrap()
    
    // Verify registration token
    let token_valid = crypto_verify_signature(
        data = concat(device_uid.bytes(), registration_token.nonce),
        sig = registration_token.backend_sig,
        sig_len = 64,
        sig_type = Ed25519,
        key = key_resolve(BACKEND_AUTH)
    )
    assert(token_valid, ERR-ID-PROV-002)
    
    // ── Phase 7: Lock Device ──
    state = PROV_LOCKING
    
    // Program security fuses
    let lock_result = identity_lock(attestation_proof)
    assert(lock_result.is_ok(), ERR-HW-FUSE-001)
    
    // Verify lock state
    let lock_state = identity_get_provisioning_state()
    assert(lock_state == LOCKED, ERR-ID-PROV-002)
    
    // Disable debug port
    debug_set_state(LOCKED)
    
    // ── Phase 8: Store Provisioning Record ──
    let provisioning_record = ProvisioningRecord {
        device_uid,
        attestation_pubkey: attestation_keypair.public_key,
        registration_token_id: registration_token.id,
        firmware_version: provisioning_data.firmware_version,
        firmware_hash: provisioning_data.firmware_hash,
        station_id: provisioning_data.station_id,
        operator_id: provisioning_data.operator_id,
        timestamp: provisioning_data.timestamp,
    }
    
    storage_write_provisioning_record(provisioning_record)
    
    // ── Phase 9: Transition to Operational ──
    state = PROV_COMPLETE
    platform_signal_provisioning_complete()
    
    return OK
```

## 8. Provisioning State Machine

```
UNINITIALIZED ──(enter provisioning)──> PROVISIONING
PROVISIONING ──(registration success)──> REGISTERED
REGISTERED ──(lock command)──> LOCKED
LOCKED ──(no further transitions)──> [Terminal]

Any state ──(failure)──> UNINITIALIZED (retry possible)
LOCKED ──(revocation)──> REVOKED
```

## 9. Provisioning Security Invariants

| Invariant | Description |
| --- | --- |
| I-PROV-1 | Key material must never leave the device in plaintext |
| I-PROV-2 | UID must be generated from TRNG (not derived, not sequential) |
| I-PROV-3 | Provisioning channel must be encrypted end-to-end |
| I-PROV-4 | Lock must be irreversible once set |
| I-PROV-5 | All provisioning actions must be logged with operator attribution |
