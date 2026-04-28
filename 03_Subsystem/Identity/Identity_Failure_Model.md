# Identity Failure Model

**Document ID:** SUB-ID-FAIL-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines all identity-related failure scenarios, their error codes, system response, and recovery paths. Identity failures affect device authentication, OTA eligibility, and backend trust.

---

## 2. Failure Scenarios

### F-ID-001: Identity Not Provisioned

**Description:** Device has never been provisioned (state = UNINITIALIZED). No UID, no KD.

**Detection:** `get_device_identity()` returns `IdentityError::NotProvisioned`.

**Response:**
1. Deny all backend access
2. Deny OTA operations
3. Device remains inoperable until provisioning

**Error Code:** `ERR-ID-IDENT-001` (Identity not provisioned)

**Recovery:** Factory provisioning required. Cannot be recovered in field.

**Test:** TEST-ID-001

---

### F-ID-002: Invalid Identity Proof

**Description:** Device presents identity proof to backend that fails verification.

**Detection:** Backend verifies `Sign(KD, UID || nonce)` → signature invalid.

**Response:**
1. Backend rejects authentication
2. Backend increments failed authentication counter
3. Rate limit applied after repeated failures

**Error Code:** `ERR-ID-AUTH-001` (Identity proof verification failed)

**Recovery:** If persistent, device may be flagged for investigation. Possible causes: KD corruption, cloning attempt, MITM.

**Test:** TEST-ID-002

---

### F-ID-003: Duplicate Identity Detected

**Description:** Backend detects two devices presenting the same UID during registration or authentication.

**Detection:** Backend dedup check: `count(DISTINCT device WHERE uid_hash = X) > 1`.

**Response:**
1. Both devices flagged as SUSPECT
2. Both authentication attempts rejected
3. SEV-1 alert generated (possible cloning attack or manufacturing fraud)
4. Investigation initiated per Incident_Response.md

**Error Code:** `ERR-ID-CLONE-002` (Backend reports duplicate UID)

**Recovery:** If genuine collision (astronomically unlikely with 128-bit TRNG): factory re-provisioning. If cloning: revoke affected devices, investigate factory.

**Test:** TEST-ID-002

---

### F-ID-004: Provisioning Interruption

**Description:** Provisioning process interrupted before completion (power loss, station failure, network error).

**Detection:** Device state ≠ LOCKED after provisioning session. Partial keys may be injected.

**Response:**
1. Device fails self-test (incomplete key set)
2. Station detects provisioning did not reach LOCK step
3. Device remains in PROVISIONING state

**Error Code:** `ERR-ID-PROV-006` (Self-test failed before lock)

**Recovery:** Restart provisioning from Step 6 (identity creation). If partial keys were injected, zeroize and restart clean.

**Test:** Factory test (power loss during provisioning)

---

### F-ID-005: Key Derivation Failure

**Description:** Device cannot derive KD or sub-keys (KD_Auth, KD_Debug, KD_Storage).

**Detection:** `derive_device_key(context)` returns crypto engine error.

**Response:**
1. Device cannot authenticate to backend
2. Device cannot establish OTA TLS session
3. Debug unlock impossible (no KD_Debug)

**Error Code:** `ERR-ID-KEY-005` (Key derivation failed)

**Recovery:** If caused by corrupted KD: factory re-provisioning required. If caused by tamper zeroization: device is irrecoverable.

**Test:** TEST-ID-001

---

### F-ID-006: Identity Storage Corruption

**Description:** Identity data (UID, state, KD handle) corrupted in secure storage.

**Detection:** `get_device_identity()` returns `ERR-ID-IDENT-002` (Identity storage corrupted).

**Response:**
1. Device enters FAILSAFE (cannot verify identity)
2. Backend access denied
3. Tamper log records storage corruption

**Error Code:** `ERR-ID-IDENT-002` (Identity storage corrupted)

**Recovery:** Factory re-provisioning. If secure element failure: hardware replacement.

**Test:** Fault injection test

---

### F-ID-007: Key Zeroization (Tamper Response)

**Description:** Keys have been zeroized due to critical tamper detection.

**Detection:** `verify_key_presence()` fails; KD handle returns "key not found".

**Response:**
1. Device enters TAMPER_LOCK (irreversible)
2. All backend access permanently denied
3. No recovery possible — device is permanently inoperative

**Error Code:** `ERR-ID-KEY-003` (KD not available — not provisioned or zeroized)

**Recovery:** None. Physical device replacement required.

**Test:** Physical pentest (trigger tamper, verify zeroization)

---

## 3. Failure Principles

| # | Principle |
| --- | --- |
| 1 | No identity → no trust → no service |
| 2 | No trust → no OTA (device cannot receive updates) |
| 3 | No recovery without secure re-provisioning (factory or field credential injection) |
| 4 | Identity failures must be auditable (logged at backend with device hash) |
| 5 | Failed authentication must not reveal whether UID exists (timing oracle prevention) |

---

## 4. References

| Document | Reference |
| --- | --- |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §7 |
| Identity Interface | `Identity_Interface.md` |
| Provisioning Flow | `Provisioning_Flow.md` |
| Anti-Cloning | `Anti_Cloning.md` |
| Incident Response | `../../06_Operations/Incident_Response.md` |
| Manufacturing Security | `../../04_Security/Manufacturing_Security.md` |
