# Attack Surface

**Document ID:** ARC-AS-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Identifies all entry points for attacks across all 11 attack tree branches (A1-A11).

---

## 2. Attack Surfaces by Domain

### 2.1 Boot Attack Surface (A1-A4)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Firmware image parsing | Malformed ImageHeader, buffer overflow | A1 (Signature Forgery) |
| Signature verification | Invalid signature, algorithm confusion | A1 (Signature Forgery) |
| Hash verification | Hash collision, length extension | A3 (Image Tampering) |
| Metadata parsing | Malformed metadata fields | A1, A3 |
| Version checking | Integer overflow, OTP manipulation | A4 (Rollback) |
| Slot selection | Forced invalid slot selection | A3 |
| MPU/MMU configuration | Misconfiguration before LOCK_BOOT | A11 (Zone Hopping) |

### 2.2 OTA Attack Surface (A2, A5)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Firmware download | MITM, malicious payload, truncation | A1, A3 |
| Update commands | Replay, spoofed commands | A2, A4 |
| Backend authentication | Certificate forgery, downgrade attack | A2 (Key Compromise) |
| Image staging | Flash wear, partial write, timing | A3 |
| Rollback mechanism | Forced rollback to vulnerable version | A4 |

### 2.3 Identity Attack Surface (A5)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| UID generation | Weak TRNG, predictable UID | A5 (Cloning) |
| Key injection | Eavesdropping, station compromise | A5 |
| Provisioning protocol | Replay, MITM during provisioning | A5 |
| Backend registration | Duplicate UID, unauthorized batch | A5 |
| Attestation proof | Forged proof, nonce reuse | A5 |

### 2.4 Crypto Attack Surface (A2, A8)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Signature verification | Timing side-channel, fault injection | A2, A8 |
| Hash computation | Length extension, collision | A8 (Side-Channel) |
| Key derivation (HKDF) | Side-channel leakage, weak inputs | A2, A8 |
| TRNG | Entropy starvation, predictable output | A2 |
| Constant-time compare | Compiler optimization breaking constant-time | A8 |

### 2.5 Supply Chain Attack Surface (A6)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Source repository | Malicious commit, branch compromise | A6 (Supply Chain) |
| Build pipeline | Toolchain compromise, dependency poisoning | A6 |
| Signing infrastructure | HSM access abuse, unauthorized signing | A6 |
| Distribution (CDN/registry) | Image substitution, metadata tampering | A6 |
| SBOM | Missing or falsified dependency data | A6 |

### 2.6 Physical Attack Surface (A7)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Debug port (JTAG/SWD) | Unauthorized debug access | A7 (Physical), A9 |
| Power supply | Voltage glitching to skip instructions | A7 |
| Clock input | Clock glitching for timing violations | A7 |
| EM emissions | Side-channel leakage (EM) | A7, A8 |
| Chip surface | Decapsulation, microprobing (SL 4) | A7 |
| Temperature | Fault injection via temperature extremes | A7 |

### 2.7 Side-Channel Attack Surface (A8)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Timing | Non-constant-time branches in crypto | A8 (Side-Channel) |
| Power (DPA/SPA) | Key-dependent power consumption | A8 |
| EM emanations | Key-dependent EM patterns | A8 |
| Cache timing | Cache line access patterns | A8 |
| Acoustic / thermal | Indirect side-channels (SL 4) | A8 |

### 2.8 Debug Attack Surface (A9)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Debug port (physical) | Unlocked debug interface | A9 (Debug) |
| Debug authentication | Brute-force, replay, protocol bypass | A9 |
| Debug commands | Memory dump, register access | A9, A2 |
| Debug lifecycle bypass | Forcing debug enable after LOCKED | A9 |
| Error messages | Information leakage via debug output | A9 |

### 2.9 Manufacturing Attack Surface (A10)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Provisioning station | Station compromise, malware | A10 (Manufacturing) |
| Station-to-device channel | Eavesdropping, MITM | A10 |
| Batch authorization | Unauthorized batch registration | A10 |
| Key injection | KD interception, weak injection protocol | A10, A5 |
| Factory network | Network intrusion during provisioning | A10 |

### 2.10 Zone Isolation Attack Surface (A11)

| Entry Point | Attack Vector | Attack Tree |
|-------------|--------------|-------------|
| Zone 2 → Zone 1 access | MPU/MMU bypass, DMA attack | A11 (Zone Hopping) |
| Shared memory | Unauthorized access before LOCK_BOOT | A11 |
| Interrupt vectors | Interrupt handler redirection | A11 |
| System calls | Privilege escalation via syscall abuse | A11 |

---

## 3. Cross-Cutting Risk Areas

| Risk Area | Affected Domains | Mitigation Class |
|-----------|-----------------|------------------|
| Parsing logic | Boot, OTA, Identity | Fuzzing, input validation |
| State transitions | All | Formal verification, exhaustive testing |
| External inputs | OTA, Provisioning, Debug | Zero-trust, verify before use |
| Timing-dependent logic | Boot, Crypto | Constant-time implementation |
| Configuration/enable bits | Boot, Debug | Redundant checks, glitch-resistant |

---

## 4. References

| Document | Reference |
|----------|-----------|
| Attack Tree | `../04_Security/Attack_Tree.md` |
| Threat Model | `Threat_Model.md` |
| Security Controls | `../04_Security/Security_Controls.md` |
| Trust Model | `Trust_Model.md` |
