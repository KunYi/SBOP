# Attack Tree Analysis

**Document ID:** SEC-AT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines structured attack paths against the SBOP system.

---

## 2. Root Goal

G-ROOT: Compromise Device Trust

---

## 3. Attack Tree

### G-ROOT: Compromise Device Trust

├── A1: Execute Unauthorized Firmware
│   ├── A1.1: Bypass Signature Verification
│   │   ├── A1.1.1: Skip verification (fault injection)
│   │   └── A1.1.2: Exploit parsing bug
│   │
│   ├── A1.2: Forge Valid Signature
│   │   ├── A1.2.1: Break crypto
│   │   └── A1.2.2: Steal signing key
│   │
│   └── A1.3: Inject Malicious Firmware via OTA
│       ├── A1.3.1: Tamper during transfer
│       └── A1.3.2: Backend compromise

---

├── A2: Break Firmware Integrity
│   ├── A2.1: Modify firmware after signing
│   └── A2.2: Exploit hash verification bug

---

├── A3: Perform Rollback Attack
│   ├── A3.1: Install old firmware
│   └── A3.2: Bypass version check

---

├── A4: Clone Device Identity
│   ├── A4.1: Extract keys
│   └── A4.2: Copy identity data

---

├── A5: Exploit OTA Mechanism
│   ├── A5.1: Interrupt update
│   └── A5.2: Corrupt storage

---

├── A6: Compromise Supply Chain
│   ├── A6.1: Compromise source code
│   │   ├── A6.1.1: Malicious commit (insider)
│   │   └── A6.1.2: Compromised developer credentials
│   ├── A6.2: Compromise build environment
│   │   ├── A6.2.1: Build server compromise
│   │   └── A6.2.2: Malicious build tool/dependency
│   ├── A6.3: Compromise signing process
│   │   ├── A6.3.1: Steal signing key from HSM
│   │   └── A6.3.2: Sign unauthorized firmware via insider
│   └── A6.4: Compromise distribution
│       ├── A6.4.1: Replace firmware on CDN/mirror
│       └── A6.4.2: Man-in-the-middle on firmware download

---

├── A7: Physical Tampering
│   ├── A7.1: PCB-level attacks
│   │   ├── A7.1.1: Bus sniffing (SPI, I2C)
│   │   └── A7.1.2: Flash memory readout (chip-off)
│   ├── A7.2: Fault injection
│   │   ├── A7.2.1: Voltage glitching on verification
│   │   ├── A7.2.2: Clock glitching on state machine
│   │   └── A7.2.3: EM fault injection on crypto
│   └── A7.3: Enclosure bypass
│       ├── A7.3.1: Open enclosure without tamper detection
│       └── A7.3.2: Bypass tamper switch

---

├── A8: Side-Channel Attack
│   ├── A8.1: Timing analysis
│   │   ├── A8.1.1: Timing oracle on signature verification
│   │   └── A8.1.2: Timing oracle on hash comparison
│   ├── A8.2: Power analysis
│   │   ├── A8.2.1: SPA on Ed25519 verification
│   │   └── A8.2.2: DPA on key derivation
│   └── A8.3: EM analysis
│       └── A8.3.1: EM leakage from crypto operations

---

├── A9: Debug Port Exploitation
│   ├── A9.1: Unauthorized debug access
│   │   ├── A9.1.1: Debug port left open after provisioning
│   │   └── A9.1.2: Debug authentication bypass
│   ├── A9.2: Debug auth credential compromise
│   │   ├── A9.2.1: Steal debug auth credentials from provisioning station
│   │   └── A9.2.2: Brute-force debug auth challenge
│   └── A9.3: Breakpoint injection during boot
│       └── A9.3.1: Inject breakpoint to skip verification

---

├── A10: Manufacturing Bypass
│   ├── A10.1: Overproduction
│   │   └── A10.1.1: Produce unauthorized devices at factory
│   ├── A10.2: Identity cloning at factory
│   │   ├── A10.2.1: Provision multiple devices with same UID
│   │   └── A10.2.2: Copy identity from one device to another
│   ├── A10.3: Key extraction during provisioning
│   │   └── A10.3.1: Intercept key injection on provisioning line
│   └── A10.4: Test mode exploitation
│       ├── A10.4.1: Ship device with test/debug still enabled
│       └── A10.4.2: Load unauthorized firmware via test mode

---

├── A11: Zone Boundary Violation (IEC 62443)
│   ├── A11.1: Zone 1 boundary breach
│   │   ├── A11.1.1: Application reads boot memory (Zone 2 → Zone 1)
│   │   └── A11.1.2: OTA writes to active slot bypassing boot verification
│   ├── A11.2: Network zone breach
│   │   ├── A11.2.1: Bypass TLS mutual authentication (Conduit A)
│   │   └── A11.2.2: Replay captured OTA traffic
│   └── A11.3: Manufacturing zone breach
│       ├── A11.3.1: Unauthorized provisioning station connects to backend
│       └── A11.3.2: Provisioning station malware exfiltrates keys

---

## 4. Logic Types

* AND: All child nodes required
* OR: Any child node sufficient

---

## 5. Security Coverage Rule

Every leaf node must have:

* Mitigation
* Requirement
* Test

## Note:

Attack execution and validation are defined in 05_Verification/Red_Team_Test_Plan.md
