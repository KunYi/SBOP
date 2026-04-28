# Functional Requirements - OTA

**Document ID:** REQ-FR-OTA-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## REQ-FR-OTA-001: Remote Firmware Update

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall support remote firmware update over a secure channel (TLS 1.3 with mutual authentication using device client certificate derived from KD_Auth). Updates shall be downloaded to the inactive slot without affecting the currently executing firmware.

**Rationale:**
Enables lifecycle management and security patching without physical access. The dual-slot model ensures the active firmware is never overwritten during update.

**Verification:** TEST (TEST-OTA-001)
**Design:** `OTA_Flow_Pseudocode.md` Phase 1-3 (AUTHENTICATE, DOWNLOAD), `Reference_Architecture.md` §6.1
**Security Goal:** G-AVAIL
**Related Requirements:** REQ-SR-OTA-001, REQ-FR-OTA-005

---

## REQ-FR-OTA-002: Firmware Verification Before Installation

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall verify the authenticity (signature) and integrity (SHA-256 hash) of downloaded firmware before marking the slot as PENDING. The verification must use the same cryptographic path as boot-time verification.

**Rationale:**
Prevents installation of compromised firmware. OTA verification is the first line of defense; boot-time verification is the second (defense in depth).

**Verification:** TEST (TEST-OTA-003)
**Design:** `OTA_Flow_Pseudocode.md` Phase 4 (VERIFY)
**Security Goal:** G-AUTH, G-INT
**Related Requirements:** REQ-FR-BOOT-001, REQ-FR-BOOT-002

---

## REQ-FR-OTA-003: Firmware Integrity During Transfer

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall ensure integrity of firmware during transfer by: (a) downloading over TLS 1.3, (b) verifying SHA-256 hash after download completes, and (c) performing write-verify on each flash chunk written. Any mismatch shall abort the update.

**Rationale:**
Network and storage corruption are realistic failure modes. TLS alone does not guarantee end-to-end integrity — the hash must be independently verified.

**Verification:** TEST (TEST-OTA-003)
**Design:** `OTA_Flow_Pseudocode.md` Phase 3 (DOWNLOAD chunk verification)
**Security Goal:** G-INT
**Related Requirements:** REQ-SR-OTA-001

---

## REQ-FR-OTA-004: Interrupted Update Recovery

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall recover from an interrupted update (power loss, network failure) without bricking. The active firmware slot shall remain unaffected. On next boot, the system shall detect the interrupted state, mark the failed slot as INVALID, and boot from the active slot.

**Rationale:**
Updates must be safe in environments where power loss is common (automotive, industrial, battery-powered devices). The device must never reach an unrecoverable state due to an update interruption.

**Verification:** TEST (TEST-OTA-002)
**Design:** `OTA_Flow_Pseudocode.md` (ota_recover_from_interruption), `OTA_Recovery.md`
**Security Goal:** G-AVAIL
**Related Requirements:** REQ-FR-BOOT-005

---

## REQ-FR-OTA-005: Maintain One Valid Image

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The system shall maintain at least one valid, bootable firmware image at all times. The active slot shall never be overwritten during OTA. The inactive slot shall not be marked ACTIVE until after verification succeeds. If both slots are compromised, the device shall enter FAILSAFE with backend recovery path.

**Rationale:**
This is the fundamental safety guarantee — no single point of failure can brick the device. The A/B dual-image model enforces this invariant.

**Verification:** TEST (TEST-OTA-002, TEST-OTA-004)
**Design:** `OTA_Flow_Pseudocode.md` Phase 5-6 (INSTALL, ACTIVATE), `Unified_State_Model.md`
**Security Goal:** G-AVAIL
**Related Requirements:** REQ-FR-BOOT-005, REQ-FR-OTA-004

---

## REQ-FR-OTA-006: OTA Rollback on Boot Failure

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
If the system fails to boot a newly installed firmware (PENDING slot does not reach ACTIVE within the confirmation window), the bootloader shall automatically fall back to the previous active slot, mark the failed slot as INVALID, and report the rollback to the backend.

**Rationale:**
Automatic rollback prevents a bad update from bricking devices in the field. Without this, every OTA carries brick risk.

**Verification:** TEST (TEST-OTA-004)
**Design:** `OTA_Flow_Pseudocode.md` (ota_rollback function), `OTA_Failure_Model.md`
**Security Goal:** G-AVAIL
**Related Requirements:** REQ-FR-BOOT-005

---

## REQ-FR-OTA-007: Update Authenticity (Backend Auth)

**Priority:** P0 (Mandatory)
**Status:** Approved

**Description:**
The device shall authenticate the OTA backend using TLS 1.3 server certificate verification. The device shall mutually authenticate to the backend using a client certificate derived from KD_Auth. Certificate validation must be strict — no self-signed or invalid certificates accepted.

**Rationale:**
Prevents MITM attacks serving malicious firmware. Mutual TLS ensures both the backend and device are who they claim to be.

**Verification:** TEST (TEST-OTA-001), RED (RED-OTA-002)
**Design:** `OTA_Flow_Pseudocode.md` Phase 1 (AUTHENTICATE)
**Security Goal:** G-AUTH
**Related Requirements:** REQ-SR-OTA-001
