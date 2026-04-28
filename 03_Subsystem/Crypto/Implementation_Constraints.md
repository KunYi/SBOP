# Cryptographic Implementation Constraints

**Document ID:** SUB-CRYPTO-IMP-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines mandatory implementation constraints for all cryptographic operations in SBOP. These constraints ensure resistance to timing side-channels, fault injection, and implementation errors. Compliance is verified by TIM-001, TIM-002, TVLA, and formal verification.

→ See `../../04_Security/Side_Channel_Countermeasures.md` for the countermeasure design rationale.
→ See `../../04_Security/Formal_Verification.md` for formal verification properties.

---

## 2. Constant-Time Requirements

### 2.1 Signature Verification

**Requirement:** Signature verification MUST execute in time independent of the signature value, key value, and message content.

```
Mandatory constraints:
  - Fixed sequence of field operations (no short-circuit on zero/invalid)
  - No data-dependent loop bounds
  - No data-dependent memory access patterns
  - No data-dependent branch conditions on secret data
  - ECDSA: scalar multiplication must use fixed-window or Montgomery ladder
  - Ed25519: verify per RFC 8032 §5.1 (fixed-time formula)

Prohibited:
  - wnaf/NAF with data-dependent window size
  - Early return on signature mismatch
  - Any timing variation correlated with correctness
```

### 2.2 Hash Comparison

**Requirement:** Hash comparison MUST use `constant_time_compare` with no early return.

```
constant_time_compare(a: &[u8], b: &[u8], len: usize) → bool:
    result = 0
    for i in 0..len:
        result |= a[i] ^ b[i]
    return result == 0

Constraints:
  - Loop always executes exactly len iterations (no break)
  - No difference in execution for matching vs non-matching at any byte position
  - Must be verified at assembly level (compiler optimization can break timing)
```

### 2.3 Version Comparison

**Requirement:** Version check MUST be fixed-sequence.

```
Constraints:
  - Compare all version fields before making decision
  - No short-circuit evaluation (must read all bytes)
  - Redundant comparison (two reads of version counter, compare results)
```

---

## 3. Memory Handling

### 3.1 Key Material

| Rule | Description |
| --- | --- |
| No plaintext exposure | Key material never in plaintext outside secure element / TEE |
| Opaque handles | Keys accessed via KeyRef handles only |
| Zeroization | All key buffers zeroized after use (memset_s or volatile zero) |
| No swap to disk | Key material never paged to flash or external memory |
| Stack clearing | Function epilogue clears local key buffers before return |

### 3.2 Sensitive Data

```
After processing sensitive data (keys, hashes, intermediate values):
  1. Overwrite buffer with zeros (use volatile_memset or equivalent)
  2. Do NOT rely on free() / drop() — they don't zero memory
  3. Compiler barriers to prevent optimization of zeroing: use secure_clear()
```

---

## 4. Error Handling

### 4.1 Uniform Error Behavior

| Rule | Description |
| --- | --- |
| No detailed errors | External interfaces return generic error codes (never "invalid R" or "bad S") |
| Timing uniformity | Error path takes same time as success path |
| No error oracle | Attacker must not be able to distinguish WHY verification failed |
| Logging restrictions | Detailed error information logged only to secure tamper log (not exposed externally) |

### 4.2 Error Categories Exposed Externally

Only these categories may be exposed to Zone 2 or backend:

| Category | Example |
| --- | --- |
| Generic verification failure | ERR-BOOT-CRYPTO-001 (signature invalid) |
| Crypto engine fault | ERR-BOOT-CRYPTO-003 (hardware accelerator error) |
| Unsupported algorithm | ERR-BOOT-CRYPTO-002 |

---

## 5. Side-Channel Resistance

### 5.1 Software-Level Countermeasures

| Countermeasure | Target | When Required |
| --- | --- | --- |
| Constant-time operations | Timing | All SL |
| Balanced code paths | Timing (SPA) | SL 2+ |
| Masking (boolean/arithmetic) | Power (DPA) | SL 3+ |
| Random delays / clock jitter | Power/EM | SL 3+ (if HW supports) |
| Pre-charge logic | Power (SPA) | SL 4 (HW-level) |

### 5.2 Hardware-Level Countermeasures

| Countermeasure | Description | Platform Requirement |
| --- | --- | --- |
| Secure element | Dedicated security processor with side-channel hardening | SL 3+ |
| Shielding | EM shielding on package | SL 4 |
| Active shield | Active mesh detecting physical probing | SL 4 |

→ See `../../04_Security/Physical_Tamper_Resistance.md`

---

## 6. Fault Injection Resistance

### 6.1 Logical Countermeasures

| Countermeasure | Description |
| --- | --- |
| Redundant verification | Signature verified twice; results compared |
| Redundant version check | Version counter read twice; values compared |
| Instruction skip detection | Check that critical instructions executed (increment counter) |
| Flow integrity check | Verify that all phases were executed (phase tracking) |

### 6.2 Hardware Countermeasures

| Countermeasure | Description |
| --- | --- |
| Glitch detection | Voltage/clock glitch monitors |
| Watchdog | Separate clock domain; must be kicked at correct flow points |

---

## 7. Determinism

| Rule | Description |
| --- | --- |
| Verification determinism | `verify_signature(data, sig, key)` must return the same result every time |
| Hash determinism | `compute_hash(data)` must produce identical output for identical input |
| No randomness in verify | Signature verification must NOT use TRNG or any randomness |

---

## 8. Compiler and Toolchain Constraints

| Constraint | Rationale |
| --- | --- |
| No optimization of security code | Critical paths compiled at -O1 or with optimization barriers |
| `volatile` for secure clear | Prevents compiler from optimizing away zeroization |
| `noinline` for crypto core | Prevents inlining that could expose timing |
| Assembly verification | Critical constant-time sections verified at assembly level |

---

## 9. References

| Document | Reference |
| --- | --- |
| Side-Channel Countermeasures | `../../04_Security/Side_Channel_Countermeasures.md` |
| Physical Tamper Resistance | `../../04_Security/Physical_Tamper_Resistance.md` |
| Formal Verification | `../../04_Security/Formal_Verification.md` |
| Crypto Algorithms | `Crypto_Algorithms.md` |
| Interface Contracts | `../../03_Subsystem/Interface_Contracts.md` |
| Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` §5 |
