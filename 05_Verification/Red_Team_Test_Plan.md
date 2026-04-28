# Red Team Test Plan

**Document ID:** VER-RED-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the offensive (red team) testing methodology for SBOP. Red team testing validates that security controls work as specified against a skilled, motivated attacker. Each test is derived from the attack tree and simulates realistic attacker capabilities at the defined security level (SL).

---

## 2. Red Team Methodology

### 2.1 Testing Principles

| Principle | Description |
| --- | --- |
| Realistic attacker | Simulate attacker with capabilities matching target SL |
| No prior knowledge (black box start) | Begin with only publicly available information |
| Full knowledge (white box) | After black box, provide full specs — can attacker exploit known design? |
| Reproducible | Each test produces repeatable results |
| Measured | Time, cost, expertise, and equipment documented |
| Defensive validation | Verify that defense works as specified, not just that attack fails |

### 2.2 Attacker Profiles

| Profile | Capability | Equipment Budget | Time Budget | Tests SL |
| --- | --- | --- | --- | --- |
| Script Kiddie | Public tools only | $0 | 1 day | SL 1 |
| Skilled Hobbyist | Custom scripts, basic HW tools | $1K | 1 week | SL 2 |
| Professional Team | Advanced tools, fault injection, side-channel | $100K | 1 month | SL 3 |
| Nation-State Lab | FIB, microprobing, custom silicon | $1M+ | 6 months | SL 4 |

---

## 3. Test Catalog

### 3.1 A1: Unauthorized Firmware Execution

#### A1.1.1: Skip Signature Verification (Fault Injection)

```
Test ID: RED-FI-001
Profile: Professional Team (SL 3)
Method:
  1. Identify signature verification code in bootloader via reverse engineering
  2. Set up voltage glitching (ChipWhisperer or custom FPGA)
  3. Time glitch to skip conditional branch after verification
  4. Attempt 1000 glitch attempts with parameter sweep
  5. If EXECUTE reached without valid signature → attack success

Expected Defense:
  - Redundant verification: even if one check is skipped, second check catches it
  - Glitch counter increments → tamper response at threshold
  - FAILSAFE entered

Success Criteria (Defense):
  - 0 successful bypasses in 1000 attempts
  - Glitch counter correctly increments
  - Tamper response triggers at configured threshold
```

#### A1.1.2: Image Header Parsing Exploit

```
Test ID: RED-FUZZ-001
Profile: Skilled Hobbyist (SL 2)
Method:
  1. Craft malformed ImageHeader variants:
     - image_size = 0
     - image_size = 0xFFFFFFFF
     - algorithm = 0xFF (invalid)
     - sig_length > image_size
     - Negative values in signed fields (if applicable)
     - Overlapping regions (payload_offset + payload_size > image_size)
  2. Attempt to boot each variant
  3. Monitor for: crash, verification bypass, unexpected state

Expected Defense:
  - All malformed headers rejected
  - Error code matches the specific validation failure
  - No crash or undefined behavior (ASAN/UBSAN clean)

Success Criteria (Defense):
  - 100% rejection rate
  - No crash, no bypass
```

#### A1.2.1: Break Cryptographic Algorithm

```
Test ID: RED-CRYPTO-001
Profile: Professional Team (SL 3)
Method:
  1. Attempt to generate valid ECDSA P-256 signature without private key
  2. Attempt to generate valid Ed25519 signature without private key
  3. Attempt hash collision: find m1 ≠ m2 where SHA-256(m1) = SHA-256(m2)
  4. Attempt length extension attack on SHA-256

Expected Defense:
  - ECDSA P-256/Ed25519 provide existential unforgeability
  - SHA-256 collision resistance prevents meaningful attacks

Success Criteria (Defense):
  - No forgery possible within time/resource budget
  - No practical SHA-256 collision found
```

#### A1.2.2: Steal Signing Key

```
Test ID: RED-KEY-001
Profile: Professional Team (SL 3)
Method:
  1. Attempt network access to HSM management interface
  2. Attempt physical access to HSM (if in scope)
  3. Attempt side-channel extraction of key during signing (power/EM)
  4. Attempt to extract key from HSM backup media
  5. Social engineering: attempt to obtain operator credentials

Expected Defense:
  - HSM network isolation
  - Physical access controls + tamper-evident seals
  - Quorum requirement (single operator cannot sign)
  - HSM designed to resist side-channel extraction

Success Criteria (Defense):
  - No key material extracted
  - All access attempts logged and alerted
```

---

### 3.2 A2: Firmware Integrity Compromise

#### A2.1: Modify Firmware After Signing

```
Test ID: RED-INT-001
Method:
  1. Take valid signed firmware image
  2. Modify 1 byte in payload region
  3. Attempt to boot modified image
  4. Repeat with modifications in: header, payload start, payload end, signature block

Expected Defense:
  - SHA-256 hash mismatch detected
  - Image rejected, slot marked INVALID
```

---

### 3.3 A3: Rollback Attack

#### A3.1: Install Old Firmware

```
Test ID: RED-ROLL-001
Method:
  1. Provision device with firmware v2
  2. Obtain validly signed firmware v1 (older)
  3. Attempt to install v1 via OTA or direct flash write
  4. Attempt to boot v1

Expected Defense:
  - Version check: v1 < current version counter → rejected
  - OTA anti-rollback rejects download

Success Criteria (Defense):
  - v1 never executes
  - Version counter never decrements
  - Error reported as ERR-BOOT-VER-001 (version too old)
```

---

### 3.4 A4: Device Cloning

#### A4.1: Extract Device Keys

```
Test ID: RED-CLONE-001
Profile: Professional Team (SL 3)
Method:
  1. Attempt to read KD from secure element via:
     a. Debug port (JTAG/SWD) — if still open
     b. Side-channel during key derivation
     c. Fault injection to bypass secure element access controls
     d. Decapsulation + microprobing (SL 4 only)
  2. If KD extracted, derive KD_Auth, KD_Debug, KD_Storage

Expected Defense:
  - Secure element / TEE prevents direct key read
  - Debug port disabled or authenticated
  - Side-channel countermeasures (masking at SL 3+)

Success Criteria (Defense):
  - No KD extraction at SL ≤ 3
```

---

### 3.5 A5: OTA Exploitation

#### A5.1: Interrupt Update (Power Loss)

```
Test ID: RED-OTA-001
Method:
  1. Initiate OTA download
  2. Cut power at each phase:
     a. During download (0%, 50%, 99%)
     b. During verification
     c. During slot write
     d. During slot status transition (PENDING write)
  3. Restore power, observe recovery

Expected Defense:
  - Download resumes from last confirmed chunk
  - Partial slot marked INVALID (CRC mismatch)
  - Device boots from previous ACTIVE slot
  - Never bricks (always reaches EXECUTE or FAILSAFE with recovery path)

Success Criteria (Defense):
  - Device recovers in all scenarios
  - No data corruption in ACTIVE slot
  - Telemetry reports interruption to backend
```

#### A5.2: Man-in-the-Middle OTA

```
Test ID: RED-OTA-002
Method:
  1. Set up MITM proxy between device and backend
  2. Attempt to serve modified firmware during download
  3. Attempt TLS downgrade attack
  4. Attempt certificate spoofing (self-signed cert for backend domain)
  5. Attempt replay of previous OTA session

Expected Defense:
  - TLS 1.3 with mutual auth (device client cert + server cert verification)
  - Image signature verified independently of TLS
  - Hash verified after download
  - Replay prevented by fresh nonce per session

Success Criteria (Defense):
  - Modified firmware rejected (signature or hash mismatch)
  - TLS downgrade rejected (TLS 1.3 minimum)
  - Spoofed cert rejected (certificate pinning or CA validation)
  - Replay rejected (nonce check)
```

---

### 3.6 A7: Physical Tampering

#### Voltage Glitching (A7.2.1)

```
Test ID: RED-FI-002
Profile: Professional Team (SL 3)
Method:
  1. Set up voltage glitching platform (ChipWhisperer Pro or equivalent)
  2. Target specific operations:
     a. Signature verification result check
     b. Version comparison
     c. Slot status transition
     d. Debug lock state
  3. Sweep glitch parameters: voltage, width, timing offset
  4. 10,000 attempts per target

Expected Defense:
  - Redundant critical checks
  - Glitch detection monitors (voltage, clock)
  - Glitch counter increments → threshold → tamper response

Success Criteria (Defense):
  - 0 successful bypasses
  - If any bypass succeeds → design change required
```

#### Enclosure Opening (A7.3.1)

```
Test ID: RED-PHY-001
Method:
  1. Attempt to open device enclosure without triggering tamper switch
  2. Attempt to disable tamper switch (short, cut, bypass)
  3. Attempt to open while device is powered off → power on

Expected Defense:
  - Tamper switch detects opening
  - Tamper response triggers (level depends on configuration)
  - Switch bypass detected (continuity check)
```

---

### 3.7 A8: Side-Channel Attack

#### Timing Oracle on Signature Verification (A8.1.1)

```
Test ID: RED-SCH-001
Profile: Professional Team (SL 3)
Method:
  1. Collect timing measurements for 10,000 signature verifications
  2. Vary: correct signatures, signatures with wrong r, wrong s, wrong length
  3. Test for timing correlation with: number of correct bytes, position of first error
  4. Attempt to construct valid signature via timing oracle

Expected Defense:
  - Constant-time verification: timing distribution identical for valid/invalid
  - TVLA t-test: |t| < 4.5 for all sample sizes

Success Criteria (Defense):
  - No timing correlation with correctness (Pearson < 0.1)
  - TVLA pass
```

#### Differential Power Analysis (A8.2.2)

```
Test ID: RED-SCH-002
Profile: Nation-State Lab (SL 4)
Method:
  1. Collect power traces during key derivation (HKDF) and ECDSA verification
  2. 100K-1M traces
  3. Apply CPA (Correlation Power Analysis) targeting KD and KI intermediates
  4. Attempt key recovery

Expected Defense (SL 3+):
  - Masking of sensitive intermediates
  - Random delays / clock jitter (if available)
  - Hardware-level countermeasures (secure element)

Success Criteria (Defense at SL 3+):
  - No key bits recovered from 1M traces
  - TVLA pass with masking enabled
```

---

### 3.8 A9: Debug Exploitation

#### Debug Left Open (A9.1.1)

```
Test ID: RED-DBG-001
Method:
  1. Attempt JTAG/SWD connection to production device
  2. Attempt to read memory, set breakpoints, single-step
  3. Attempt to extract keys via debug

Expected Defense:
  - Debug port locked (fuse blown)
  - Connection refused
  - No memory access

Success Criteria (Defense):
  - Debug connection fails
  - No memory or register access
```

#### Debug Auth Brute Force (A9.2.2)

```
Test ID: RED-DBG-002
Method:
  1. Attempt debug authentication with random KD_Debug values
  2. Attempt 10 consecutive failures
  3. After lockout, attempt with correct KD_Debug

Expected Defense:
  - Rate limiting (exponential backoff)
  - Permanent lock at 10 failures
  - Correct credentials rejected after lockout

Success Criteria (Defense):
  - Lockout triggered at exactly 10 failures
  - Lockout is permanent and irreversible
  - No timing oracle on challenge-response (constant-time comparison)
```

---

### 3.9 A11: Zone Boundary Violation

#### Application Reads Boot Memory (A11.1.1)

```
Test ID: RED-ZONE-001
Method:
  1. From Zone 2 (application), attempt to read:
     a. Bootloader code region
     b. Key storage region
     c. Slot metadata (write attempt)
  2. Use raw pointer dereference, DMA, or peripheral access

Expected Defense:
  - MPU/MMU fault on unauthorized access
  - Application crashes (Memory Manage Fault) or blocked
  - Zone 1 memory never readable from Zone 2

Success Criteria (Defense):
  - All unauthorized accesses blocked
  - MPU fault handler invoked
  - Zone 1 integrity maintained
```

---

## 4. Test Environment Requirements

### 4.1 Required Equipment

| Test Category | Equipment | Approximate Cost |
| --- | --- | --- |
| Fault injection | ChipWhisperer Pro, oscilloscope, EM probe | $5K-$30K |
| Side-channel | Oscilloscope (1 GHz+), EM probe, power analysis board | $20K-$100K |
| Debug testing | J-Link Ultra+, logic analyzer | $2K-$10K |
| Network testing | Managed switch with mirror port, TLS proxy | $1K-$5K |
| Physical testing | Thermal camera, X-ray (for packaging analysis) | $5K-$50K |

### 4.2 Test Device Quantities

| Test | Devices Needed |
| --- | --- |
| Fault injection (destructive) | 20-50 devices |
| Side-channel | 5-10 devices |
| Debug testing | 5 devices |
| OTA testing | 10 devices |
| Physical tamper | 5-10 devices |

---

## 5. Reporting

### 5.1 Test Report Template

```
Red Team Test Report
====================
Test ID: RED-XXX-XXX
Attack Tree Node: AX.Y.Z
Attacker Profile: [Script Kiddie / Skilled Hobbyist / Professional / Nation-State]
Date: YYYY-MM-DD
Tester: [Name]

Method:
  [Detailed description of attack method]

Results:
  Attempts: N
  Successes: 0
  Defenses triggered: [list]

Conclusion:
  PASS: Attack unsuccessful, defense functioned as specified
  FAIL: Attack succeeded → file bug, re-design defense
  INCONCLUSIVE: Equipment limitation, time budget exceeded → note in risk register

Evidence:
  - Oscilloscope traces: [files]
  - Logic analyzer capture: [files]
  - Device logs: [files]
```

---

## 6. Success Criteria Summary

| Criteria | Threshold |
| --- | --- |
| No attack achieves root goal (execute unauthorized code, extract keys, clone device) | 100% |
| All attacks lead to safe state (FAILSAFE, TAMPER_LOCK, or rejection) | 100% |
| All tamper events logged | 100% |
| All test results reproducible | 100% |

---

## 7. References

| Document | Reference |
| --- | --- |
| Attack Tree | `../04_Security/Attack_Tree.md` |
| Fault Injection Test | `Fault_Injection_Test.md` |
| Fuzzing Strategy | `Fuzzing_Strategy.md` |
| Physical Tamper Resistance | `../04_Security/Physical_Tamper_Resistance.md` |
| Side-Channel Countermeasures | `../04_Security/Side_Channel_Countermeasures.md` |
| Secure Debug Architecture | `../04_Security/Secure_Debug_Architecture.md` |
| Zone Conduit Model | `../04_Security/Zone_Conduit_Model.md` |
