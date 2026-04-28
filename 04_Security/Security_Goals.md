# Security Goals

**Document ID:** SEC-GOAL-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Core Security Goals

### 1.1 Authenticity (G-AUTH)

Only trusted, authorized firmware shall execute.

**Enforced by:**
- Cryptographic signature verification (ECDSA P-256 / Ed25519)
- Root of trust anchored in KR during provisioning
- All images must chain to KR through key hierarchy

→ See `01_Requirements/REQ-FR-BOOT-001`, `REQ-SR-BOOT-001`

---

### 1.2 Integrity (G-INT)

Firmware shall be protected against unauthorized modification.

**Enforced by:**
- SHA-256 hash verification of firmware payload
- Constant-time hash comparison
- Image format validation before any execution

→ See `01_Requirements/REQ-FR-BOOT-002`, `REQ-SR-BOOT-002`

---

### 1.3 Availability (G-AVAIL)

The system shall remain recoverable under all failure conditions.

**Enforced by:**
- Dual-image A/B slot model
- OTA rollback on activation failure
- FAILSAFE state with backend recovery path
- Recovery boot mode for on-orbit/in-field recovery

→ See `01_Requirements/REQ-FR-OTA-004`, `REQ-FR-OTA-005`

---

### 1.4 Anti-Cloning (G-CLONE)

Each device shall have a unique, non-transferable cryptographic identity.

**Enforced by:**
- Unique Device ID (UID) from TRNG during provisioning
- Device Key (KD) derived from root key (KR) + UID
- Backend validation rejects duplicate UIDs
- Identity locked irreversibly after provisioning

→ See `01_Requirements/REQ-SR-ID-001`, `REQ-SR-ID-002`

---

### 1.5 Confidentiality (G-CONF)

Cryptographic keys shall never be exposed outside their protection boundary.

**Enforced by:**
- Key isolation in secure element or TEE
- Key references only (never expose raw key material)
- Key zeroization on tamper detection
- No key access via debug port at any access level

→ See `01_Requirements/REQ-SR-CONF-001`

---

## 2. Extended Security Goals

### 2.1 Supply Chain Provenance (G-SC)

Every firmware artifact shall be traceable from source code to deployed binary.

**Enforced by:**
- Reproducible builds with bit-identical verification
- SBOM generated at build time (SPDX / CycloneDX)
- Commit signing and code provenance
- HSM-backed firmware signing
- Signed firmware distribution with per-device verification

→ See `01_Requirements/REQ-SR-SC-001`, `REQ-SR-SC-002`
→ See `04_Security/Supply_Chain_Security.md`

---

### 2.2 Physical Tamper Resistance (G-PHY)

The system shall detect and respond to physical tampering attempts.

**Enforced by:**
- Tamper detection sensors (enclosure, voltage, clock, temperature)
- Redundant verification checks for fault injection defense
- Key zeroization on critical tamper detection
- Tamper-evident enclosure and secure element integration
- FAILSAFE state on tamper confirmation

→ See `01_Requirements/REQ-SR-PHY-001`, `REQ-SR-PHY-002`
→ See `04_Security/Physical_Tamper_Resistance.md`

---

### 2.3 Side-Channel Resistance (G-SCH)

Cryptographic operations shall not leak secret information through physical side-channels.

**Enforced by:**
- Constant-time implementation of all crypto operations
- Constant-time memory comparison for all security-critical checks
- No data-dependent branches or memory access patterns
- Timing side-channel measurement and TVLA testing
- Power analysis countermeasures (masking for SL 3+)

→ See `01_Requirements/REQ-SR-SCH-001`
→ See `04_Security/Side_Channel_Countermeasures.md`

---

### 2.4 Debug Security (G-DBG)

Debug interfaces shall not provide unauthorized access to device assets.

**Enforced by:**
- Debug port disabled or authenticated before operational mode
- Challenge-response authentication using device-specific KD_Debug
- Rate-limited authentication attempts with lockout
- Authorized debug unlock with time-limited backend-signed tokens
- Key material always protected regardless of debug access level

→ See `01_Requirements/REQ-SR-DBG-001`, `REQ-SR-DBG-002`
→ See `04_Security/Secure_Debug_Architecture.md`

---

### 2.5 Manufacturing Integrity (G-MFR)

Only authorized devices shall be provisioned, and all provisioning shall be auditable.

**Enforced by:**
- Factory physical and logical access controls
- Encrypted key injection during provisioning
- Batch authorization with size and time limits
- Device lifecycle tracking with immutable provisioning record
- Backend registration verification (no duplicates, no overproduction)

→ See `01_Requirements/REQ-SR-MFR-001`, `REQ-SR-MFR-002`
→ See `04_Security/Manufacturing_Security.md`

---

## 3. Goal-to-Requirement Trace

| Goal | Requirement | Design | Test |
| --- | --- | --- | --- |
| G-AUTH | REQ-SR-BOOT-001 | Boot_Overview, Crypto_Algorithms | TEST-BOOT-002 |
| G-INT | REQ-SR-BOOT-002 | Image_Format, Crypto_Algorithms | TEST-BOOT-003 |
| G-AVAIL | REQ-FR-OTA-004/005 | OTA_Recovery, System_Flow | TEST-OTA-002/004 |
| G-CLONE | REQ-SR-ID-001/002 | Identity_Overview, Anti_Cloning | TEST-ID-002 |
| G-CONF | REQ-SR-CONF-001/002 | Crypto_Overview, Key_Hierarchy | ANALYSIS |
| G-SC | REQ-SR-SC-001/002 | Supply_Chain_Security | SC-AUDIT-001/002 |
| G-PHY | REQ-SR-PHY-001/002 | Physical_Tamper_Resistance | FI-001 |
| G-SCH | REQ-SR-SCH-001 | Side_Channel_Countermeasures | TIM-001/002, TVLA |
| G-DBG | REQ-SR-DBG-001/002 | Secure_Debug_Architecture | DBG-TEST-001/002 |
| G-MFR | REQ-SR-MFR-001/002 | Manufacturing_Security | Factory audit |
