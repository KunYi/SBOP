# SBOP Test Cases Specification

**Document ID:** VER-TC-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines concrete, executable test cases for SBOP verification. Each test includes setup, steps, expected results, pass/fail criteria, and traceability to requirements.

---

## 2. Boot Test Cases

### TEST-BOOT-001: Valid Firmware Boot (Normal Path)

**Requirement:** REQ-FR-BOOT-001, REQ-FR-BOOT-003, REQ-FR-BOOT-005
**Priority:** P0

**Setup:**
1. Flash validly signed firmware to Slot A
2. Slot A metadata: status=ACTIVE, CRC valid
3. Slot B: EMPTY
4. Version counter in OTP: 0
5. Firmware version: 1

**Steps:**
1. Assert RESET
2. Monitor boot state transitions via debug UART (if available) or GPIO toggles
3. Wait for application entry point reached (heartbeat GPIO or UART message)

**Expected Results:**
- State transitions: RESET→INIT→SELECT_SLOT→LOAD_IMAGE→VERIFY_SIGNATURE→VERIFY_INTEGRITY→CHECK_VERSION→COMMIT_VERSION→MARK_ACTIVE→LOCK_BOOT→EXECUTE
- No error codes logged
- Application executes (heartbeat active)
- Boot duration within timing budget
- Zone 1 memory locked (MPU fault on access attempt from Zone 2)

**Pass Criteria:** Application executes, no error codes, all phases confirmed.

---

### TEST-BOOT-002: Invalid Signature Rejection

**Requirement:** REQ-FR-BOOT-001, REQ-SR-BOOT-001
**Priority:** P0

**Setup:**
1. Flash firmware with VALID signature to Slot A (ACTIVE)
2. Flip 1 bit in signature block (byte 63 of signature)
3. CRC-32 on SlotMetadata recalculated and updated

**Steps:**
1. Assert RESET
2. Monitor boot flow

**Expected Results:**
- State reaches VERIFY_SIGNATURE → verification fails
- Transition to FAILSAFE (not EXECUTE)
- Slot A marked INVALID
- Error code: ERR-BOOT-CRYPTO-001
- If Slot B has valid fallback → boot from Slot B
- If no fallback → device remains in FAILSAFE

**Pass Criteria:** Application does NOT execute. Error code correct. Slot correctly marked.

---

### TEST-BOOT-003: Corrupted Firmware Rejection

**Requirement:** REQ-FR-BOOT-002
**Priority:** P0

**Setup:**
1. Flash validly signed firmware to Slot A
2. Corrupt 1 byte in payload region (not header, not signature)
3. Recalculate SlotMetadata CRC

**Steps:**
1. Assert RESET
2. Monitor boot flow

**Expected Results:**
- VERIFY_SIGNATURE passes (signature over original hash)
- VERIFY_INTEGRITY: SHA-256 recomputed → mismatch detected
- Transition to FAILSAFE
- Error code: ERR-BOOT-CRYPTO-005

**Pass Criteria:** Application does NOT execute.

---

### TEST-BOOT-004: Rollback Rejection

**Requirement:** REQ-FR-BOOT-004
**Priority:** P0

**Setup:**
1. Flash firmware v2 to Slot A, successfully boot
2. Version counter in OTP now = 2
3. Flash firmware v1 (validly signed, but older) to Slot B
4. Attempt to boot Slot B

**Steps:**
1. Set boot slot selection to B
2. Assert RESET
3. Monitor boot flow

**Expected Results:**
- CHECK_VERSION: v1 (1) ≤ counter (2) → rejection
- Transition to FAILSAFE
- Error code: ERR-BOOT-VER-001
- v1 does NOT execute

**Pass Criteria:** v1 rejected. Version counter remains 2 (not decremented).

---

### TEST-BOOT-005: All Verification Phases Enforced

**Requirement:** REQ-FR-BOOT-003, REQ-FR-BOOT-005
**Priority:** P0

**Setup:**
Standard boot with valid image.

**Steps (for each phase):**
1. Skip phase by directly setting state to next phase (simulate fault injection)
2. Attempt EXECUTE
3. Repeat for all mandatory phases

**Expected Results:**
- Phase tracking detects skipped phase
- EXECUTE blocked
- FAILSAFE entered
- Error code: ERR-BOOT-STATE-001

**Pass Criteria:** No path from RESET to EXECUTE that skips any phase.

---

## 3. OTA Test Cases

### TEST-OTA-001: Successful Update

**Requirement:** REQ-FR-OTA-001, REQ-FR-OTA-002, REQ-FR-OTA-007
**Priority:** P0

**Setup:**
1. Device running firmware v1 in Slot A (ACTIVE), Slot B EMPTY
2. Backend has firmware v2 available for this device
3. TLS certificates valid
4. Network connectivity OK

**Steps:**
1. Trigger OTA check (manual or scheduled)
2. Device authenticates to backend (mutual TLS)
3. Device requests update → backend returns UpdateInfo for v2
4. Device downloads v2 to Slot B (chunked transfer)
5. Device verifies signature and hash
6. Device sets Slot B to PENDING
7. Device reboots
8. Boot verifies Slot B → marks ACTIVE
9. Application confirms boot success

**Expected Results:**
- Authentication succeeds
- Download completes without error
- Hash matches expected
- Signature verifies
- Boot succeeds from Slot B
- Slot B is now ACTIVE, Slot A is INACTIVE (fallback)
- Backend receives boot confirmation

**Pass Criteria:** Device running v2, all verification checks passed.

---

### TEST-OTA-002: Interrupted Download Recovery

**Requirement:** REQ-FR-OTA-004
**Priority:** P0

**Setup:**
Same as TEST-OTA-001.

**Steps:**
1. Start download of v2
2. At 50% download progress: cut power
3. Restore power
4. Device boots from Slot A (still ACTIVE)
5. OTA client detects interrupted state
6. OTA client resumes download or restarts

**Expected Results:**
- After power restoration: boot succeeds from Slot A (unaffected)
- OTA client detects Slot B in inconsistent state
- Slot B metadata CRC mismatch → Slot B marked INVALID
- OTA client erases Slot B, restarts download from byte 0
- Full download completes, verification passes, update proceeds normally

**Pass Criteria:** Device recovers without user intervention, update eventually succeeds.

---

### TEST-OTA-003: Reject Bad Firmware During OTA

**Requirement:** REQ-FR-OTA-002, REQ-FR-OTA-003
**Priority:** P0

**Setup:**
Same as TEST-OTA-001 but backend serves firmware with bit-flipped payload.

**Steps:**
1. Trigger OTA
2. Download completes
3. Hash verification runs

**Expected Results:**
- Hash mismatch detected: ERR-OTA-DOWNLOAD-002
- Image discarded
- Slot B erased
- Device remains on v1 in Slot A
- Error reported to backend

**Pass Criteria:** Bad firmware never installed. Device unaffected.

---

### TEST-OTA-004: Automatic Rollback on Boot Failure

**Requirement:** REQ-FR-OTA-006, REQ-FR-OTA-005
**Priority:** P0

**Setup:**
1. Device running v1 (Slot A ACTIVE)
2. Install v2_bad (corrupted signature) to Slot B
3. Slot B marked PENDING
4. Device reboots

**Steps:**
1. Boot attempts Slot B (PENDING)
2. Signature verification fails
3. Boot marks Slot B INVALID
4. Boot selects Slot A as fallback
5. Boot boots v1 from Slot A
6. Application reports rollback to backend

**Expected Results:**
- v1 executes (fallback successful)
- Slot B status = INVALID
- fallback_count incremented
- Backend receives rollback notification with error code
- Device is operational (not bricked)

**Pass Criteria:** Device running v1, rollback logged.

---

## 4. Identity Test Cases

### TEST-ID-001: Unique Identity Provisioning

**Requirement:** REQ-SR-ID-001
**Priority:** P0

**Setup:**
1. Unprovisioned device (state = UNINITIALIZED)
2. Provisioning station connected
3. Backend available

**Steps:**
1. Enter provisioning mode
2. Generate UID from TRNG
3. Derive KD from KR + UID
4. Inject KD into secure element
5. Lock identity
6. Verify lock (attempt to modify → must fail)
7. Register with backend

**Expected Results:**
- UID generated with ≥ 128 bits entropy
- KD injected and responsive to challenge
- Identity locked (immutable after LOCK)
- Backend registration succeeds
- UID is unique (no collision)

**Pass Criteria:** Device identity provisioned, locked, registered. Provisioning record in backend audit log.

---

### TEST-ID-002: Duplicate Identity Rejection

**Requirement:** REQ-SR-ID-002
**Priority:** P0

**Setup:**
1. Device A provisioned with UID_X
2. Attempt to provision Device B with same UID_X (or clone identity from A)

**Steps:**
1. Device B attempts registration with UID_X
2. Backend dedup check runs

**Expected Results:**
- Backend detects duplicate UID
- Registration rejected
- Error: ERR-ID-CLONE-002
- SEV-1 alert generated
- Both devices flagged

**Pass Criteria:** Duplicate rejected, alert generated.

---

### TEST-ID-003: Provisioning With Interruption Recovery

**Requirement:** REQ-SR-MFR-001
**Priority:** P0

**Steps:**
1. Start provisioning
2. Cut power at various points (after UID gen, after KD inject, before lock)
3. Restore power
4. Observe recovery

**Expected Results (per interruption point):**
- Before key inject: restart from UID generation
- After partial key inject: zeroize, restart clean
- Before lock: re-verify keys, complete lock
- After lock: device is LOCKED, provisioning complete

**Pass Criteria:** Device always reaches a clean state (LOCKED or ready to re-provision).

---

## 5. Crypto Test Cases

### TEST-CRYPTO-001: Signature Verification Correctness

**Requirement:** REQ-SR-BOOT-001
**Priority:** P0

**Test Vectors:**
- Valid Ed25519 signatures (RFC 8032 test vectors)
- Valid Ed25519 signatures (RFC 8032 test vectors)
- Invalid signatures: wrong r, wrong s, r=0, s=0, r=n, s=n
- Truncated signatures
- Signatures with wrong algorithm tag
- Signatures over wrong data

**Pass Criteria:** All valid signatures accepted. All invalid signatures rejected. No crash.

---

### TEST-CRYPTO-002: Hash Consistency

**Requirement:** REQ-FR-BOOT-002
**Priority:** P0

**Test Vectors:**
- SHA-256("") = e3b0c442...
- SHA-256("abc") = ba7816bf...
- SHA-256 of 1 MB zero block
- SHA-256 of 1 MB random data (compare with OpenSSL)

**Pass Criteria:** All hashes match reference implementation. Deterministic (same output every time).

---

## 6. Negative Test Cases

| # | Test | Input | Expected |
| --- | --- | --- | --- |
| NEG-001 | Malformed ImageHeader | magic ≠ 0x53424F50 | Rejected, ERR-BOOT-IMAGE-001 |
| NEG-002 | Invalid algorithm ID | algorithm = 0xFF | Rejected, ERR-BOOT-CRYPTO-002 |
| NEG-003 | image_size = 0 | Header field | Rejected, ERR-BOOT-IMAGE-001 |
| NEG-004 | image_size > flash size | Header field | Rejected, ERR-BOOT-IMAGE-001 |
| NEG-005 | Truncated image | image_size > actual data | Rejected, ERR-BOOT-STOR-002 |
| NEG-006 | Corrupted metadata CRC | SlotMetadata | Rejected, ERR-BOOT-STOR-003 |
| NEG-007 | Missing SignatureBlock | sig_length = 0 | Rejected, ERR-BOOT-CRYPTO-001 |
| NEG-008 | Both slots INVALID | Both slots bad | FAILSAFE, backend recovery |

---

## 7. References

| Document | Reference |
| --- | --- |
| Test Strategy | `Test_Strategy.md` |
| Fault Injection Test | `Fault_Injection_Test.md` |
| Red Team Test Plan | `Red_Team_Test_Plan.md` |
| Fuzzing Strategy | `Fuzzing_Strategy.md` |
| Coverage Model | `Coverage_Model.md` |
| Error Code Catalog | `../02_System_Design/Error_Code_Catalog.md` |
