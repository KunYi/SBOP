# SBOP Error Code Catalog

**Document ID:** SYS-ERR-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document is the central registry of all error codes in the SBOP system. Each error code has a unique identifier, a defined severity level, a description, and a prescribed handling behavior.

---

## 2. Error Code Format

```
ERR-<COMPONENT>-<CATEGORY>-<ID>
```

| Component | Prefix | Description |
| --- | --- | --- |
| Boot | ERR-BOOT | Boot subsystem errors |
| Crypto | ERR-CRYPTO | Cryptographic subsystem errors |
| OTA | ERR-OTA | Update subsystem errors |
| Identity | ERR-ID | Identity subsystem errors |
| Storage | ERR-STOR | Storage abstraction errors |
| Parse | ERR-PARSE | Parsing and format errors |
| Hardware | ERR-HW | Hardware/platform errors |
| Security | ERR-SEC | Cross-cutting security errors |

**Category examples:** CRYPTO, PARSE, STATE, AUTH, STORAGE, NETWORK, VERSION, KEY, INTEGRITY

---

## 3. Error Severity Levels

| Severity | Code | Description | Default Handling |
| --- | --- | --- | --- |
| FATAL | F | System cannot continue; enter FAILSAFE | → FAILSAFE; no recovery without external intervention |
| ERROR | E | Operation failed; system can recover | → Retry or rollback depending on context |
| WARNING | W | Non-critical anomaly detected | → Log; continue operation |
| INFO | I | Informational event | → Log only |

---

## 4. Boot Subsystem Errors

### 4.1 Crypto Verification Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-BOOT-CRYPTO-001 | FATAL | Signature verification failed | Firmware signature does not verify with the expected key | FAILSAFE | OTA recovery via backend |
| ERR-BOOT-CRYPTO-002 | FATAL | Hash mismatch | Computed SHA-256 of payload does not match header.hash | FAILSAFE | OTA recovery via backend |
| ERR-BOOT-CRYPTO-003 | FATAL | Unsupported signature algorithm | signature_type in signature block is unknown or unsupported | FAILSAFE | Firmware re-sign with supported algorithm |
| ERR-BOOT-CRYPTO-004 | FATAL | Key verification failed | Key slot revoked (SR1 revocation bitmap), public key fingerprint mismatch against SR1 OTP, or (legacy) verification key reference resolved to no key | FAILSAFE | Key rotation via backend (non-revoked slot) or re-provisioning |
| ERR-BOOT-CRYPTO-005 | FATAL | HW/SW hash mismatch | HW SHA-256 result does not match software SHA-256 (possible HW HASH glitch or FIH attack) | FAILSAFE | Reboot; if persistent: hardware fault |
| ERR-BOOT-CRYPTO-006 | FATAL | Crypto engine initialization failed | Hardware crypto accelerator not responding | FAILSAFE | Hardware replacement |
| ERR-BOOT-CRYPTO-007 | FATAL | IV reuse detected | Same (KD, IV_dev) pair used twice — possible AES-CTR keystream replay attack | FAILSAFE | Investigate tamper; re-flash with new image |

### 4.2 Parse Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-BOOT-PARSE-001 | FATAL | Invalid image magic | header.magic != 0x53424F50 ("SBOP") | FAILSAFE | Re-flash with valid image |
| ERR-BOOT-PARSE-002 | FATAL | Unsupported header version | header.header_version > MAX_SUPPORTED_VERSION | FAILSAFE | Bootloader update required |
| ERR-BOOT-PARSE-003 | FATAL | Invalid header size | header.header_size < MIN_HEADER_SIZE | FAILSAFE | Image corruption; re-flash |
| ERR-BOOT-PARSE-004 | FATAL | Invalid image size | header.image_size == 0 or exceeds slot capacity | FAILSAFE | Image corruption; re-flash |
| ERR-BOOT-PARSE-005 | FATAL | Malformed signature block | signature_size invalid or signature data misaligned | FAILSAFE | Re-sign image |
| ERR-BOOT-PARSE-006 | FATAL | Invalid signature block pointer | Signature block offset points outside image | FAILSAFE | Image corruption |
| ERR-BOOT-PARSE-007 | FATAL | Reserved fields not zero | header.reserved[] or reserved2[] contains non-zero bytes | FAILSAFE | Re-flash with valid image |
| ERR-BOOT-PARSE-008 | FATAL | Board ID mismatch | header.board_id does not match device board ID | FAILSAFE | Flash correct SKU firmware |
| ERR-BOOT-PARSE-009 | FATAL | Encrypted key index mismatch | flags bits 12-15 enc_key_index != header.enc_key_index | FAILSAFE | Image corruption; re-flash |
| ERR-BOOT-PARSE-010 | FATAL | Header HMAC verification failed | HMAC-SHA-256-128(KD, header[0..56]) != header.header_hmac | FAILSAFE | SPI bus tampering suspected; re-flash |

### 4.3 State Machine Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-BOOT-STATE-001 | FATAL | Invalid state transition | Attempted transition not allowed in state machine | FAILSAFE | Reset; if persistent: hardware fault |
| ERR-BOOT-STATE-002 | FATAL | State machine integrity check failed | State variable CRC/parity mismatch (possible fault injection) | FAILSAFE | Reboot; tamper investigation |
| ERR-BOOT-STATE-003 | FATAL | Redundant check mismatch | Two verification paths disagreed (possible fault injection) | FAILSAFE | Reboot; tamper investigation |
| ERR-BOOT-STATE-004 | FATAL | No bootable slot | Both slots are INVALID, CORRUPT, or EMPTY | FAILSAFE | Serial recovery or re-provision |
| ERR-BOOT-STATE-005 | ERROR | Unconfirmed image reverted | Slot was in TESTING state — application did not call boot_set_confirmed() before reset | Immediate rollback to fallback | Application must confirm health after update |
| ERR-BOOT-STATE-006 | ERROR | Confirm called on non-TESTING slot | boot_set_confirmed() called but slot is already ACTIVE or in another state | Return error to caller | Application should check slot status first |

### 4.4 Version Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-BOOT-VERSION-001 | FATAL | Rollback detected | header.firmware_version < stored_version_counter | FAILSAFE | Flash newer firmware |
| ERR-BOOT-VERSION-002 | FATAL | Version counter corrupted | Stored version counter cannot be read or is invalid | FAILSAFE | Re-flash and re-provision |
| ERR-BOOT-VERSION-003 | ERROR | Zero version | header.firmware_version == 0 (may indicate unsigned image) | FAILSAFE | Sign with proper version |
| ERR-BOOT-VERSION-004 | FATAL | Version below factory floor | header.firmware_version < min_allowed_version (SR1 OTP) | FAILSAFE | Flash factory or newer firmware |
| ERR-BOOT-VERSION-005 | FATAL | Version skip detected | firmware_version != stored_version + 1 (skipped versions in OTP counter) | FAILSAFE | Check signing key integrity |
| ERR-BOOT-VERSION-006 | FATAL | Version exceeds ceiling | firmware_version > max_allowed_version (metadata ceiling) | FAILSAFE | Verify backend authorization; possible signing key compromise |

---

## 5. Cryptographic Subsystem Errors

### 5.1 Signature Verification Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-CRYPTO-SIG-001 | ERROR | Invalid signature format | Signature does not conform to expected DER/raw encoding | Return error to caller | Re-sign with correct format |
| ERR-CRYPTO-SIG-002 | ERROR | Signature too large | signature_size exceeds maximum for algorithm | Return error to caller | Verify signature block format |
| ERR-CRYPTO-SIG-003 | ERROR | Invalid public key format | Key data does not parse as expected curve point | Return error to caller | Verify key provisioning |

### 5.2 Hash Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-CRYPTO-HASH-001 | ERROR | Hash computation failed | Internal hash engine error | Return error to caller | Retry; if persistent: hardware fault |
| ERR-CRYPTO-HASH-002 | ERROR | Unsupported hash algorithm | Requested algorithm not available | Return error to caller | Use SHA-256 only |

### 5.3 Key Derivation Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-CRYPTO-KDF-001 | ERROR | Key derivation failed | HKDF internal error | Return error to caller | Retry; if persistent: provisioning error |
| ERR-CRYPTO-KDF-002 | ERROR | Insufficient entropy for KDF | Input key material too short or low-entropy | Return error to caller | Check key source |
| ERR-CRYPTO-KDF-003 | ERROR | Invalid KDF salt | Salt format or length invalid | Return error to caller | Check KDF parameters |

### 5.4 RNG Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-CRYPTO-RNG-001 | FATAL | TRNG failure | Hardware random number generator failed health tests | FAILSAFE (no crypto possible) | Hardware replacement |
| ERR-CRYPTO-RNG-002 | WARNING | TRNG health test warning | RNG output failed continuous health test | Retry RNG read | If persistent: upgrade to FATAL |

---

## 6. OTA Subsystem Errors

### 6.1 Authentication Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-OTA-AUTH-001 | ERROR | Device authentication failed | Backend rejected device authentication | Retry (max 3); then alert | Verify device registration |
| ERR-OTA-AUTH-002 | ERROR | Backend certificate invalid | TLS certificate chain validation failed | Abort update | Check system time; network issue |
| ERR-OTA-AUTH-003 | ERROR | Update not authorized | Backend denied update for this device/version | Abort update | May be policy; no action needed |

### 6.2 Download Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-OTA-DOWNLOAD-001 | ERROR | Download interrupted | Connection lost during firmware transfer | Retry from checkpoint if supported; else restart | Resume or restart download |
| ERR-OTA-DOWNLOAD-002 | ERROR | Download hash mismatch | Received data hash does not match expected | Discard; retry download | Retry from clean start |
| ERR-OTA-DOWNLOAD-003 | ERROR | Insufficient storage | Inactive slot cannot fit downloaded image | Abort; report to backend | Backend should check before authorizing |

### 6.3 Installation Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-OTA-INSTALL-001 | ERROR | Write to inactive slot failed | Storage write error during firmware install | Retry write; if persistent: mark slot bad | Use other slot if available |
| ERR-OTA-INSTALL-002 | ERROR | Slot verification failed | Written data does not match expected after write | Retry write; if persistent: mark slot bad | Use other slot |
| ERR-OTA-INSTALL-003 | ERROR | Active slot overwrite attempted | Bug: tried to write to currently active slot | Abort; internal error | Software bug; report |

### 6.4 Activation Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-OTA-ACTIVATE-001 | ERROR | Pending image signature invalid | Image in pending slot fails boot-time signature check | Boot reverts to previous slot | OTA rollback |
| ERR-OTA-ACTIVATE-002 | ERROR | Pending image hash invalid | Image in pending slot fails boot-time hash check | Boot reverts to previous slot | OTA rollback |

### 6.5 Rollback Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-OTA-ROLL-001 | ERROR | Rollback triggered | Boot failed on new image; reverting to previous | Automatic rollback; report to backend | Verify previous image integrity |

---

## 7. Identity Subsystem Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-ID-PROV-001 | ERROR | Provisioning already completed | Device in LOCKED state; provisioning rejected | Return error; device operational | None needed |
| ERR-ID-PROV-002 | ERROR | Provisioning data invalid | Received provisioning data does not validate | Abort provisioning | Retry provisioning |
| ERR-ID-PROV-003 | ERROR | Provisioning station auth failed | Station authentication during provisioning failed | Abort provisioning | Verify station credentials |
| ERR-ID-ATST-001 | ERROR | Attestation failed | Device cannot generate valid attestation proof | Return error | Possible cloning attempt |
| ERR-ID-CLONE-001 | ERROR | Duplicate identity detected | Backend reports UID already registered | Alert; enter FAILSAFE | Investigate cloning |
| ERR-ID-CLONE-002 | ERROR | Clone attempt suspected | Backend anomaly detection flagged identity | Alert; require re-authentication | Investigate |

### 7.2 Authentication Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-ID-AUTH-001 | ERROR | Device authentication failed | KD_Auth challenge-response failed | Retry; if persistent: revoke | Check device identity |

### 7.3 Identity Storage Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-ID-IDENT-001 | FATAL | Identity not provisioned | Device in UNINITIALIZED state | Reject all operations | Provision device |
| ERR-ID-IDENT-002 | FATAL | Identity storage corrupted | UID or KD stored data fails integrity check | FAILSAFE | Re-provision device |

### 7.4 Key Access Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-ID-KEY-003 | FATAL | KD not available | Device key zeroized or never provisioned | FAILSAFE | Re-provision |
| ERR-ID-KEY-005 | ERROR | Key derivation failed | Crypto engine error during HKDF derive | Retry; if persistent: FAILSAFE | Check crypto engine |

### 7.5 Extended Provisioning Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-ID-PROV-004 | ERROR | Key verification failed | Injected key challenge-response mismatch | Abort provisioning | Retry provisioning |
| ERR-ID-PROV-005 | ERROR | Lock failed | Hardware error during identity lock | Abort; device rejected | Replace device |
| ERR-ID-PROV-006 | ERROR | Self-test failed before lock | Device self-test failed during provisioning | Abort provisioning | Retry; if persistent: reject device |

---

## 8. Storage Abstraction Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-STOR-READ-001 | ERROR | Storage read failed | Hardware error during flash read | Retry; if persistent: FAILSAFE | Hardware replacement |
| ERR-STOR-WRITE-001 | ERROR | Storage write failed | Hardware error during flash write | Retry; if persistent: mark sector bad | Remap if wear-leveling supported |
| ERR-STOR-CORRUPT-001 | FATAL | Metadata corruption | Image slot metadata unreadable or invalid | FAILSAFE | Re-flash all slots |
| ERR-STOR-ERASE-001 | ERROR | Sector erase failed | Flash sector erase timeout or failure | Retry; mark bad | Remap |

---

## 9. Hardware / Platform Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-HW-TAMPER-001 | FATAL | Tamper detected | Physical tamper event triggered | Key zeroization; FAILSAFE | Physical replacement |
| ERR-HW-TAMPER-002 | WARNING | Tamper warning | Suspicious event detected (glitch counter increasing) | Log; alert backend | Monitor; upgrade to FATAL if persistent |
| ERR-HW-DEBUG-001 | WARNING | Debug access attempt | Debug connection attempted while LOCKED | Log; if repeated: upgrade severity | Investigate |
| ERR-HW-FUSE-001 | FATAL | Fuse programming failed | Security fuse could not be programmed during provisioning | Abort provisioning; device rejected | Replace device |
| ERR-HW-RESET-001 | WARNING | Unexpected reset | System reset from non-boot source | Log; if repeated: investigate | Check watchdog, power supply |
| ERR-HW-NOR-001 | FATAL | NOR not detected / wrong chip | JEDEC manufacturer ID, type, or capacity mismatch at INIT | FAILSAFE | Check SPI bus; replace NOR chip |
| ERR-HW-NOR-002 | FATAL | NOR BP bits unrecoverable | Block Protect bits could not be set on NOR flash — possible hardware fault or attack | FAILSAFE | Hardware replacement |

---

## 10. Serial Recovery Protocol Errors

| Error ID | Severity | Description | Detail | Handling | Recovery |
| --- | --- | --- | --- | --- | --- |
| ERR-SRP-AUTH-001 | ERROR | Authentication required | Command requires authenticated session | Return error to host | Host must send AUTH_CHALLENGE + AUTH_RESPONSE |
| ERR-SRP-AUTH-002 | ERROR | Session expired | Auth session timeout elapsed | Return error to host | Host must re-authenticate |
| ERR-SRP-AUTH-003 | ERROR | Brute-force lockout active | Too many failed auth attempts | Return error to host; include retry-after hint | Host must wait for lockout period |
| ERR-SRP-FRAME-001 | ERROR | Invalid frame | Bad magic (not 0x5352) or CRC-16 mismatch | Discard frame silently | Host must re-sync and retransmit |
| ERR-SRP-FRAME-002 | ERROR | Frame too large | Declared length > 2048 bytes | Discard frame | Host must fragment payload |
| ERR-SRP-FRAME-003 | ERROR | Sequence mismatch | Frame sequence number out of order | Discard frame; do not NAK (prevents ACK storm) | Host re-syncs on timeout |
| ERR-SRP-CMD-001 | ERROR | Unknown opcode | Received opcode is not recognized | Return error to host | Host must check protocol version |
| ERR-SRP-CMD-002 | ERROR | Invalid state | Command not permitted in current protocol state | Return error to host | Host must check device state first |

---

## 11. Error Handling Rules

### 11.1 General Rules

| Rule | Description |
| --- | --- |
| ERR-RULE-001 | FATAL errors MUST enter FAILSAFE within bounded time (< 1ms from detection) |
| ERR-RULE-002 | ERROR responses to external callers MUST NOT leak internal state |
| ERR-RULE-003 | All errors MUST be logged with error ID, timestamp, and context |
| ERR-RULE-004 | Error codes returned across trust boundaries must be sanitized (no stack/internals) |
| ERR-RULE-005 | All error paths must be tested (→ `05_Verification/Test_Cases.md`) |

### 11.2 Error Propagation Across Subsystems

| From → To | Propagation Rule |
| --- | --- |
| Crypto → Boot | All crypto errors propagate as ERR-BOOT-CRYPTO-XXX (mapped) |
| Storage → Boot | Storage errors propagate as ERR-BOOT-PARSE-XXX or ERR-STOR-XXX |
| OTA → Boot | OTA activation errors propagate as ERR-BOOT-CRYPTO-XXX |
| Identity → OTA | Identity errors propagate as ERR-OTA-AUTH-XXX |

---

## 12. Error Code Allocation Table

| Subsystem | Error Range | Available Codes | Used Codes |
| --- | --- | --- | --- |
| Boot: CRYPTO | ERR-BOOT-CRYPTO-001..099 | 99 | 6 |
| Boot: PARSE | ERR-BOOT-PARSE-001..099 | 99 | 10 |
| Boot: STATE | ERR-BOOT-STATE-001..099 | 99 | 3 |
| Boot: VERSION | ERR-BOOT-VERSION-001..099 | 99 | 6 |
| Crypto: SIG | ERR-CRYPTO-SIG-001..099 | 99 | 3 |
| Crypto: HASH | ERR-CRYPTO-HASH-001..099 | 99 | 2 |
| Crypto: KDF | ERR-CRYPTO-KDF-001..099 | 99 | 3 |
| Crypto: RNG | ERR-CRYPTO-RNG-001..099 | 99 | 2 |
| OTA: AUTH | ERR-OTA-AUTH-001..099 | 99 | 3 |
| OTA: DOWNLOAD | ERR-OTA-DOWNLOAD-001..099 | 99 | 3 |
| OTA: INSTALL | ERR-OTA-INSTALL-001..099 | 99 | 3 |
| OTA: ACTIVATE | ERR-OTA-ACTIVATE-001..099 | 99 | 2 |
| OTA: ROLL | ERR-OTA-ROLL-001..099 | 99 | 1 |
| Identity: PROV | ERR-ID-PROV-001..099 | 99 | 6 |
| Identity: AUTH | ERR-ID-AUTH-001..099 | 99 | 1 |
| Identity: IDENT | ERR-ID-IDENT-001..099 | 99 | 2 |
| Identity: KEY | ERR-ID-KEY-001..099 | 99 | 2 |
| Identity: ATST | ERR-ID-ATST-001..099 | 99 | 1 |
| Identity: CLONE | ERR-ID-CLONE-001..099 | 99 | 2 |
| Storage | ERR-STOR-001..099 | 99 | 4 |
| Hardware | ERR-HW-001..099 | 99 | 7 |
| SRP: AUTH | ERR-SRP-AUTH-001..099 | 99 | 3 |
| SRP: FRAME | ERR-SRP-FRAME-001..099 | 99 | 3 |
| SRP: CMD | ERR-SRP-CMD-001..099 | 99 | 2 |
