# Provisioning Operations

**Document ID:** OPS-PROV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the manufacturing and provisioning operations for device onboarding: how devices are initialized with cryptographic identity, registered with the backend, and transitioned from the manufacturing floor to operational deployment.

---

## 2. Provisioning Environment

### 2.1 Facility Requirements

| Requirement | Specification |
| --- | --- |
| Physical access control | Badge + biometric; visitor escort required |
| Network isolation | Provisioning network has no internet access; air-gapped from corporate network |
| Station hardening | Minimal OS; no unnecessary services; read-only root filesystem |
| Environmental monitoring | Camera coverage of all provisioning stations; video retained 90 days |
| EMI shielding | Provisioning area shielded from external RF (prevents remote fault injection) |

### 2.2 Station Requirements

| Requirement | Specification |
| --- | --- |
| Station authentication | Mutual TLS with backend; station-specific client certificate |
| Station attestation | Remote attestation (TPM or similar) before provisioning session |
| Station whitelist | Backend only accepts connections from known station identities |
| Session limits | Each station authorized for specific batch size and time window |
| Key handling | KD, KD_Debug derived in secure element or TEE on station; never written to disk |
| Audit logging | All station operations logged to backend (non-repudiable) |

---

## 3. Provisioning Flow

### 3.1 Phase 1: Pre-Provisioning

```
Step 1: Station authenticates to backend (mutual TLS + station attestation)
Step 2: Backend verifies station is in whitelist and within authorized window
Step 3: Backend issues provisioning session token (time-limited, batch-scoped)
Step 4: Station loads provisioning firmware (if not pre-flashed)
Step 5: Station verifies provisioning firmware signature
```

### 3.2 Phase 2: Identity Creation

```
Step 6:  Device powers on, enters PROVISIONING state
Step 7:  Device samples TRNG → UID (128-bit, verified entropy per NIST SP 800-90B)
Step 8:  Station derives KD = HKDF(KR, UID) using HSM-backed KR handle
Step 9:  Station derives KD_Debug = HKDF(KD, "DEBUG")
Step 10: Station derives KD_Auth = HKDF(KD, "AUTH")
Step 11: Station injects KD, KD_Debug, KD_Auth into device secure element
Step 12: Device confirms key injection (challenge-response verification)
```

### 3.3 Phase 3: Registration

```
Step 13: Device generates attestation proof: Sign(KD, UID || timestamp || nonce)
Step 14: Station forwards attestation + UID + batch ID to backend
Step 15: Backend validates:
         a. UID not previously registered (dedup check)
         b. Batch is within authorized size
         c. Station is authorized for this batch
Step 16: Backend registers device: UID, KD hash, batch ID, station ID, timestamp
Step 17: Backend returns registration confirmation (signed)
Step 18: Device verifies backend signature, stores registration confirmation
```

### 3.4 Phase 4: Lock and Verify

```
Step 19: Device runs self-test:
         a. Signature verification works (test vector)
         b. Hash verification works (test vector)
         c. TRNG health tests pass
         d. Secure element accessible and responsive
         e. Tamper sensors functional
Step 20: Device locks identity: sets LOCKED flag (irreversible)
Step 21: Device disables debug port (fuse blown or permanent lock)
Step 22: Device verifies debug port is disabled (readback check)
Step 23: Device transitions to FACTORY_DONE state
Step 24: Station logs provisioning completion to backend
Step 25: Device powers down, ready for shipment
```

---

## 4. Batch Authorization

### 4.1 Batch Parameters

| Parameter | Description | Example |
| --- | --- | --- |
| Batch ID | Unique identifier for this production batch | BATCH-2026-0428-001 |
| Authorized quantity | Maximum devices in this batch | 10,000 |
| Station ID | Authorized provisioning station | STATION-SZX-03 |
| Time window | Validity period for batch authorization | 2026-04-28 08:00-20:00 UTC |
| Firmware version | Version being provisioned | 2.1.0 |
| Operator ID | Factory operator responsible | OP-4721 |

### 4.2 Oversubscription Prevention

```
Backend enforces:
  count(registered_devices_in_batch) ≤ batch.authorized_quantity

If exceeded:
  - Registration rejected for excess devices
  - Alert generated (possible overproduction)
  - Batch flagged for audit
```

---

## 5. Failure Handling

### 5.1 Failure Categories

| Category | Examples | Action |
| --- | --- | --- |
| Recoverable | Transient network error, station restart | Retry from last successful step |
| Device-rejectable | TRNG failure, secure element failure, self-test failure | Reject device; mark for decommissioning |
| Station-rejectable | Station attestation failure, batch window expired | Halt station; investigate |
| Backend-rejectable | UID collision (astronomically unlikely), batch limit exceeded | Reject registration; investigate |

### 5.2 Rejected Device Decommissioning

```
If device fails provisioning:
  1. Record failure reason with device serial number (if any)
  2. If keys were injected before failure:
     a. Send key zeroization command
     b. Verify zeroization
  3. Mark device as REJECTED in backend (if registered)
  4. Physically segregate device from passed devices
  5. Batch rejected devices for failure analysis
  6. Destroy devices that cannot be zeroized
```

---

## 6. Provisioning Audit Trail

### 6.1 Audit Events

| Event | Data Logged |
| --- | --- |
| Station session start | Station ID, timestamp, batch ID, attestation result |
| Device UID generation | UID hash, TRNG health test result, timestamp |
| Key injection | Key type (KD, KD_Debug, KD_Auth), injection method, verification result |
| Backend registration | UID hash, batch ID, station ID, timestamp, registration ID |
| Device lock | Lock confirmation, debug disable confirmation, timestamp |
| Provisioning failure | Failure step, error code, device identifier, timestamp |
| Batch completion | Batch ID, total attempted, total passed, total failed, timestamp |

### 6.2 Audit Retention

| Record Type | Retention Period |
| --- | --- |
| Provisioning audit events | 10 years (or device lifetime + 2 years) |
| Station attestation logs | 5 years |
| Batch authorization records | 10 years |
| Failure analysis reports | 5 years |
| Video surveillance | 90 days |

---

## 7. Station Security Procedures

### 7.1 Daily Station Setup

```
1. Operator authenticates to station (badge + PIN)
2. Station runs self-attestation (TPM quote)
3. Station authenticates to backend (mutual TLS)
4. Backend validates station health (attestation, patch level, antivirus)
5. Backend issues daily session token
6. Operator loads batch authorization
7. Station enters provisioning-ready state
```

### 7.2 Station Decommissioning

```
When station is retired or repurposed:
  1. Export all audit logs to backend
  2. Cryptographic erase of station storage
  3. TPM cleared
  4. Station certificate revoked in backend
  5. Physical destruction of station storage media
  6. Decommission record filed
```

---

## 8. Multi-Factory Operations

### 8.1 Factory Authorization

| Factory | Location | Authorized Products | KR Access |
| --- | --- | --- | --- |
| Primary factory | Shenzhen, CN | All products | HSM with KR handle (derive-only) |
| Secondary factory | Penang, MY | All products | HSM with KR handle (derive-only) |
| Pilot line | Munich, DE | Engineering samples only | Separate test KR (not production) |

### 8.2 Cross-Factory Controls

- Each factory has its own KR handle — no KR material shared between factories
- Batch authorizations are factory-specific
- Backend detects cross-factory UID collisions (should never happen with TRNG)
- Factory production volumes audited quarterly against backend registrations

---

## 9. Test-to-Production Transition

```
Development keys (test KR, test KI):
  - Used during development and testing
  - Never used to sign production firmware
  - Test KR has different HKDF info prefix than production KR
  - Test keys stored in separate HSM partition

Production keys (production KR, production KI):
  - Used only for production devices
  - Generated in separate HSM partition or separate HSM
  - Access requires production quorum (different operators than dev)

Transition check (final provisioning step):
  - Device verifies it has production KR-derived KD (not test)
  - Test-derived KD would fail backend registration
```

---

## 10. References

| Document | Reference |
| --- | --- |
| Manufacturing Security | `../04_Security/Manufacturing_Security.md` |
| Identity Overview | `../03_Subsystem/Identity/Identity_Overview.md` |
| Provisioning Flow | `../03_Subsystem/Identity/Provisioning_Flow.md` |
| Key Management | `Key_Management.md` |
| Secure Debug Architecture | `../04_Security/Secure_Debug_Architecture.md` |
| Data Structures | `../02_System_Design/Data_Structures.md` §DeviceID, ProvisioningData |
