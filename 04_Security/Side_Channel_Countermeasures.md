# Side-Channel Attack Countermeasures

**Document ID:** SEC-SCH-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the architectural requirements for side-channel attack resistance in SBOP. It covers timing analysis, power analysis, and electromagnetic (EM) emanations analysis — the three most relevant side-channel vectors for embedded secure boot systems.

---

## 2. Side-Channel Threat Model

### 2.1 Side-Channel Attack Classes

| Attack Class | Measurement | Information Leaked | Typical Countermeasure |
| --- | --- | --- | --- |
| **Timing Analysis** | Execution time variation | Key bits, branch decisions, data values | Constant-time algorithms |
| **Simple Power Analysis (SPA)** | Power trace shape | Operation type, key-dependent branches | Fixed-sequence operations, balancing |
| **Differential Power Analysis (DPA)** | Statistical correlation over many power traces | Key bits from sub-threshold correlations | Masking, blinding, random delays |
| **EM Analysis** | EM emissions pattern | Similar to power analysis, plus spatial info | Shielding, noise generation |
| **Cache Timing** | Cache hit/miss timing | Memory access patterns | Cache-partitioning, constant-time |
| **Acoustic Analysis** | Sound from components | Very low bandwidth; impractical for SBOP | Not a primary concern |

### 2.2 Attacker Profiles

| Profile | Capability | Measurement Access |
| --- | --- | --- |
| Remote timing attacker | Measure response time over network | OTA protocol timing only |
| Local non-invasive | Oscilloscope, EM probe | Device exterior (power, EM) |
| Local semi-invasive | Chip decapsulation, microprobing | Die-level measurements |

### 2.3 Threat Assessment for SBOP

| Operation | Side-Channel Sensitivity | Rationale |
| --- | --- | --- |
| ECDSA signature verification | **High** | Public key operations can leak key information via timing |
| SHA-256 hash computation | **Medium** | Hash is public; but data-dependent timing can leak firmware content |
| Key derivation (HKDF) | **High** | Key material is processed; leaks could expose derived keys |
| Constant-value comparison | **High** | memcmp timing creates an oracle for byte-by-byte attacks |
| State machine transitions | **Low** | State values are public by design |
| Image header parsing | **Low** | Header contents are authenticated, not secret |

---

## 3. Timing Attack Countermeasures

### 3.1 Constant-Time Requirement

All cryptographic operations and security-critical comparisons in SBOP must be constant-time with respect to secret data:

| Requirement | Description |
| --- | --- |
| SCH-TIM-001 | Signature verification must execute in constant time regardless of signature validity |
| SCH-TIM-002 | Hash computation must be constant-time regardless of input data |
| SCH-TIM-003 | Key derivation (HKDF) must be constant-time regardless of input key material |
| SCH-TIM-004 | Memory comparison for authentication values (hashes, signatures) must use constant-time comparison |

### 3.2 Constant-Time Comparison

```
// REQUIRED pattern: constant-time memory comparison
// Execution time depends ONLY on length (public), NOT on content
function constant_time_compare(a: &[u8], b: &[u8], len: usize) -> bool:
    if len == 0:
        return true
    
    result: u8 = 0
    for i in 0..len:
        result = result | (a[i] ^ b[i])
    
    return result == 0
```

### 3.3 Forbidden Patterns

| Pattern | Reason | Replacement |
| --- | --- | --- |
| `memcmp(a, b, n)` | Returns early on first difference (timing oracle) | `constant_time_compare` |
| `if result == 0: return OK else: return ERR` | Branch on secret data | `let status = 0 - constant_time_compare(...) ; return status_map[status]` |
| Data-dependent loop bounds | Loop count leaks data | Fixed iteration count (worst-case) |
| Lookup tables indexed by secret data | Cache timing side-channel | Constant-time table lookup or bit-sliced implementation |

---

## 4. Power Analysis Countermeasures

### 4.1 SPA Countermeasures

| Requirement | Description |
| --- | --- |
| SCH-SPA-001 | Crypto operations must use fixed instruction sequence (no data-dependent branches) |
| SCH-SPA-002 | Key-dependent operations must not create distinguishable power signatures |
| SCH-SPA-003 | Public-key operations (ECDSA verify) should use balanced point addition/doubling |

### 4.2 DPA Countermeasures

| Requirement | Description |
| --- | --- |
| SCH-DPA-001 | For SL 3+: Key material in internal operations must be masked (XOR with random mask, then unmask result) |
| SCH-DPA-002 | HKDF operations should apply blinding where practical |
| SCH-DPA-003 | Random delays or dummy operations may be inserted to decorrelate power traces (with care: do not create new timing channels) |

### 4.3 Hardware-Level Power Countermeasures

SBOP may specify interface requirements for hardware power countermeasures:

| Interface Requirement | Description |
| --- | --- |
| SCH-HW-PWR-001 | Platform should support power supply filtering/decoupling to reduce signal-to-noise ratio |
| SCH-HW-PWR-002 | Platform may support on-chip power scrambling / noise generation |

---

## 5. EM Analysis Countermeasures

| Requirement | Description |
| --- | --- |
| SCH-EM-001 | For SL 3+: Device enclosure should provide EM shielding (conductive coating or mesh) |
| SCH-EM-002 | PCB layout should minimize EM emissions from crypto-relevant traces (route sensitive signals on inner layers) |
| SCH-EM-003 | Decoupling capacitors near MCU power pins to reduce conducted emissions |

---

## 6. Side-Channel Resistant Crypto API

### 6.1 API Contract

All SBOP crypto subsystem functions must satisfy:

```
Contract: Constant-Time Execution
    For all functions processing secret or security-relevant data:
        - Execution cycle count must be independent of input values
        - Branch decisions must not depend on secret data
        - Memory access patterns must not depend on secret data
        - Violation: timing test will fail; must be re-implemented
```

### 6.2 Verified Functions

| Function | Sensitivity | Verification Method |
| --- | --- | --- |
| `crypto_verify_signature(data, sig, key)` | High (timing of verification may leak whether sig is close to valid) | Timing measurement over N=100k inputs |
| `crypto_compute_hash(data)` | Medium (data is public; but integrity requires determinism) | Determinism check |
| `crypto_constant_time_compare(a, b)` | High (used for hash/signature comparison) | Timing measurement over all byte positions |
| `crypto_derive_key(ctx, input)` | High (key material processed) | Timing + DPA measurement |
| `crypto_random_bytes(len)` | Low (output is random) | Statistical tests; not side-channel relevant |

---

## 7. Verification of Side-Channel Resistance

### 7.1 Test Vector Leakage Assessment (TVLA)

SBOP's crypto implementation must pass TVLA (based on ISO 17825 / FIPS 140-3 non-invasive testing):

| Test | Description | Pass Criteria |
| --- | --- | --- |
| Fixed vs. Random (FvR) | Compare power traces for fixed key vs. random key | t-test < 4.5 threshold |
| Fixed vs. Fixed (FvF) | Compare power traces for two fixed keys | t-test < 4.5 threshold |
| Timing uniformity | Measure cycle count over N=100k random inputs | Max-min ≤ 2 cycles |

### 7.2 Timing Measurement

| Test | Description | Method |
| --- | --- | --- |
| TIM-001 | Verify constant-time compare | Measure cycle count for each byte position (0..31); all must be equal |
| TIM-002 | Verify constant-time signature verify | Measure cycle count for valid, invalid-early, invalid-late signatures |
| TIM-003 | Verify constant-time hash | Measure cycle count for zero-filled, random, structured inputs |

### 7.3 Test Environment Requirements

- Oscilloscope: ≥ 500 MHz bandwidth, ≥ 2 GS/s sample rate (for power/EM)
- Power measurement: Shunt resistor or EM probe at MCU VDD
- EM measurement: Near-field H-field probe
- Statistical software: Welch's t-test computation over ≥ 10k traces

---

## 8. Error Handling and Side-Channels

Error handling must not create timing oracles:

| Requirement | Description |
| --- | --- |
| SCH-ERR-001 | Error codes returned to caller must not distinguish between "signature invalid" and "signature malformed" (same timing, same error code) |
| SCH-ERR-002 | Authentication failure must not reveal whether the failure was due to invalid identity vs. invalid credential |
| SCH-ERR-003 | All verification failures must take the same execution path and timing |

---

## 9. Crypto Agility and Side-Channels

Future cryptographic algorithm changes must maintain side-channel resistance:

| Guideline | Description |
| --- | --- |
| SCH-AGL-001 | Any new algorithm added to SBOP must be assessed for side-channel leakage before approval |
| SCH-AGL-002 | Post-quantum algorithms (when adopted) require new side-channel analysis (different operations, different leakage models) |
| SCH-AGL-003 | Algorithm deprecation must consider whether existing protections apply to the replacement |

---

## 10. Requirements Trace

| Side-Channel Attack | SBOP Requirement | Countermeasure | Verification |
| --- | --- | --- | --- |
| Timing analysis (signature verify) | REQ-SR-SCH-001, SCH-TIM-001 | Constant-time verification | TIM-001, TIM-002 |
| Timing analysis (hash compare) | REQ-SR-SCH-001, SCH-TIM-004 | constant_time_compare | TIM-001 |
| SPA on crypto operations | SCH-SPA-001 through SCH-SPA-003 | Fixed-sequence, balanced ops | TVLA (FvR) |
| DPA on key derivation | SCH-DPA-001, SCH-DPA-002 | Masking, blinding | TVLA (FvR) |
| EM analysis | SCH-EM-001 through SCH-EM-003 | Shielding, PCB layout | EM TVLA |
| Timing oracle via error codes | SCH-ERR-001 through SCH-ERR-003 | Uniform error handling | Code review + timing test |

---

## 11. Residual Risk

Side-channel resistance is inherently statistical; no system is completely immune:

| Residual Risk | Accepted? | Justification |
| --- | --- | --- |
| High-trace-count DPA (> 1M traces) | Accepted for SL ≤ 2 | Requires extended physical access and expertise |
| Professional EM analysis | Accepted for SL ≤ 2 | Mitigated by shielding at SL 3+ |
| Remote timing attacks on OTA | Mitigated | Constant-time verification; network jitter adds noise |
| Microprobing (die-level) | Accepted for SL ≤ 3 | Requires FIB/semiconductor lab resources |
