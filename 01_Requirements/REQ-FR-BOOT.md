# Functional Requirements - Boot

**Document ID:** REQ-FR-BOOT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## REQ-FR-BOOT-001: Firmware Authenticity Verification

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall verify the authenticity of the firmware image before execution using cryptographic signature verification (Ed25519).

**Rationale:**
Prevents execution of unauthorized firmware. This is the single most critical security requirement — bypass enables all other attacks.

**Verification:** TEST (TEST-BOOT-002)
**Design:** `Boot_Flow_Pseudocode.md` Phase 4 (VERIFY_SIGNATURE), `Crypto_Algorithms.md` §2
**Security Goal:** G-AUTH
**Related Requirements:** REQ-SR-BOOT-001, REQ-SR-SCH-001

---

## REQ-FR-BOOT-002: Firmware Integrity Verification

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall verify the integrity of the firmware image before execution using SHA-256 hash verification with constant-time comparison.

**Rationale:**
Detects corruption (accidental or malicious) of firmware in storage or during transfer. Without integrity verification, modified firmware could execute if signature check is flawed.

**Verification:** TEST (TEST-BOOT-003)
**Design:** `Boot_Flow_Pseudocode.md` Phase 5 (VERIFY_INTEGRITY), `Crypto_Algorithms.md` §3
**Security Goal:** G-INT
**Related Requirements:** REQ-SR-BOOT-002

---

## REQ-FR-BOOT-003: Verification Failure Response

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall prevent execution of firmware that fails any verification step (authenticity, integrity, or version). On any verification failure, the system shall transition to the FAILSAFE state and shall not execute the unverified firmware.

**Rationale:**
Ensures fail-closed behavior. A verification failure at any stage must prevent execution — no partial verification.

**Verification:** TEST (TEST-BOOT-001, TEST-BOOT-005)
**Design:** `Boot_Flow_Pseudocode.md` (handle_boot_error function), `Boot_Failure_Model.md`
**Security Goal:** G-AUTH, G-INT
**Related Requirements:** REQ-FR-BOOT-005

---

## REQ-FR-BOOT-004: Anti-Rollback Version Check

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall verify that the firmware version is strictly greater than the current monotonic version counter stored in OTP or write-protected storage. Firmware with version ≤ counter shall be rejected.

**Rationale:**
Prevents an attacker from reinstalling an older, validly signed but vulnerable firmware version (rollback attack). The version counter must be hardware-protected against decrement.

**Verification:** TEST (TEST-BOOT-004), FAULT (FI-004)
**Design:** `Boot_Flow_Pseudocode.md` Phase 6-7 (CHECK_VERSION, COMMIT_VERSION), `Unified_State_Model.md`
**Security Goal:** G-AUTH (anti-rollback sub-property)
**Related Requirements:** REQ-SR-OTA-002

---

## REQ-FR-BOOT-005: FAILSAFE State Transition

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall transition to a FAILSAFE state upon any unrecoverable verification failure. In FAILSAFE, no application firmware shall execute, and the device shall await backend-initiated recovery or fallback to a known-good slot.

**Rationale:**
FAILSAFE is the ultimate safety barrier — the device must be inert rather than running untrusted code. Recovery from FAILSAFE must be possible without physical access.

**Verification:** TEST (TEST-BOOT-001, TEST-BOOT-005)
**Design:** `Boot_Flow_Pseudocode.md` (handle_boot_error), `Boot_Failure_Model.md` §F-BOOT-001..005
**Security Goal:** G-AVAIL (recovery path)
**Related Requirements:** REQ-FR-OTA-004, REQ-FR-OTA-005

---

## REQ-FR-BOOT-006: Redundant Verification (SL 3+)

**Priority:** P1 (Required for SL 3+)
**Status:** Proposed

**Description:**
For SL 3+, critical verification steps (signature verification, version comparison) shall be performed redundantly with independent execution paths. If redundant results differ, the system shall treat this as a tamper event and enter FAILSAFE.

**Rationale:**
Defense against fault injection attacks that attempt to skip or corrupt a single verification operation.

**Verification:** FAULT (FI-001, FI-004)
**Design:** `Boot_Flow_Pseudocode.md`, `Physical_Tamper_Resistance.md` §7.2
**Security Goal:** G-PHY
**Related Requirements:** REQ-SR-PHY-001

---

## REQ-FR-BOOT-007: Phase Execution Verification

**Priority:** P1 (Required for SL 3+)
**Status:** Proposed

**Description:**
The boot process shall track which phases have been executed and shall refuse to enter EXECUTE if any mandatory phase was skipped. Phase tracking must be stored in a fault-resistant manner (redundant flags or encoded state).

**Rationale:**
Prevents fault injection that skips entire verification phases. The state machine must be self-auditing.

**Verification:** FAULT (FI-001)
**Design:** `Boot_Flow_Pseudocode.md` §10 (state transition table)
**Security Goal:** G-AUTH
**Related Requirements:** REQ-SR-PHY-002
