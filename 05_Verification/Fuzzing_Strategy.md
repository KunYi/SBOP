# Fuzzing Strategy

**Document ID:** VER-FUZZ-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the fuzz testing strategy for all SBOP input attack surfaces. Fuzzing discovers vulnerabilities in parsers, protocol handlers, and state machines by feeding malformed, unexpected, or random inputs and verifying that the system fails safely.

---

## 2. Fuzzing Targets

### 2.1 Primary Targets

| Target | Input Surface | Risk | Fuzzing Priority |
| --- | --- | --- | --- |
| Image header parser | `ImageHeader` struct (64 bytes) | Critical — bypass verification | P0 |
| Signature block parser | `SignatureBlock` struct | Critical — signature forgery | P0 |
| OTA metadata parser | `UpdateInfo` struct | High — OTA bypass | P1 |
| Slot metadata parser | `SlotMetadata` struct | High — slot corruption | P1 |
| Debug auth challenge parser | `DebugAuthChallenge` struct | High — debug unlock | P1 |
| Provisioning data parser | `ProvisioningData` struct | High — identity spoofing | P1 |
| Telemetry record parser | `TelemetryRecord` struct | Medium — backend DoS | P2 |

### 2.2 Secondary Targets

| Target | Input Surface | Priority |
| --- | --- | --- |
| OTA download chunk handler | Variable-length binary | P1 |
| Boot state machine inputs | Control flags, error codes | P0 |
| Crypto API parameter validation | Key handles, buffer pointers | P1 |

---

## 3. Fuzzing Methodology

### 3.1 Mutation-Based Fuzzing

**Tool:** AFL++, libFuzzer, or Honggfuzz

```
Approach:
  1. Seed corpus: valid ImageHeader, SignatureBlock, UpdateInfo, etc.
  2. Mutate: bit flips, byte insertion/deletion, arithmetic mutations
  3. Feed mutated input to parser under test
  4. Monitor for: crash, hang, undefined state, memory error
  5. Triage: categorize failures, deduplicate, prioritize
```

### 3.2 Generation-Based (Grammar) Fuzzing

**Tool:** libFuzzer with custom mutator, or Grammarinator

```
Approach:
  1. Define input grammar from Data_Structures.md struct definitions
  2. Generate inputs that are structurally valid but semantically invalid:
     - Valid magic bytes, invalid length fields
     - Valid structure, impossible field combinations
     - Maximum and minimum boundary values for all fields
     - Type confusion (e.g., algorithm ID for version field)
  3. Generate inputs targeting specific parser paths
```

### 3.3 State-Aware Fuzzing

**Tool:** Custom harness + libFuzzer

```
Approach:
  1. Model parser as state machine (from pseudocode)
  2. Track which states are reached for each input
  3. Guide mutation to target uncovered states
  4. Goal: cover all parser states and transitions
```

---

## 4. Fuzzing Harnesses

### 4.1 Image Header Fuzzer

```
Harness: fuzz_image_header(input_bytes)
  1. Cast input_bytes → ImageHeader (if ≥ 64 bytes)
  2. Validate all header fields per Image_Format.md §3
  3. Check: no crash, no OOB read, error returned for invalid input
  4. Check: magic = 0x53424F50 is enforced (no bypass)
  5. Check: image_size bounds enforced
  6. Check: algorithm field validated against known algorithms
  7. Check: reserved fields handled correctly (must be zero or ignored)
```

### 4.2 Signature Block Fuzzer

```
Harness: fuzz_signature_block(input_bytes)
  1. Parse SignatureBlock from input
  2. Verify signature format per algorithm
  3. Check: no crash, no memory corruption
  4. Check: invalid signature length rejected
  5. Check: signature overflow/underflow handled
```

### 4.3 OTA Metadata Fuzzer

```
Harness: fuzz_ota_metadata(input_bytes)
  1. Parse UpdateInfo from input
  2. Validate all fields
  3. Check: download_url bounds and format validation
  4. Check: version field extremes (0, MAX_U32)
  5. Check: image_size vs actual payload consistency
  6. Check: hash field format validation
```

### 4.4 State Machine Fuzzer

```
Harness: fuzz_boot_state_machine(input_sequence)
  1. Initialize state machine to RESET
  2. Feed sequence of events/inputs
  3. After each event: check state is valid (not undefined)
  4. Check: no transition bypasses required states
  5. Check: no path from RESET to EXECUTE skipping VERIFY_SIG, VERIFY_INT, CHECK_VER
```

---

## 5. Mutation Strategies

### 5.1 Bit-Level Mutations

| Mutation | Description | Example |
| --- | --- | --- |
| Bit flip | Flip 1-32 random bits | 0x53424F50 → 0x53424F51 |
| Byte set | Set random byte to interesting value | 0x00, 0xFF, 0x7F, 0x80 |
| Byte increment/decrement | ±1 on random byte | Length field manipulation |
| Block copy | Copy bytes from one region to another | Confuse payload with header |
| Block insert/delete | Insert or remove N bytes | Buffer over/underflow |

### 5.2 Semantic Mutations

| Mutation | Target | Description |
| --- | --- | --- |
| Length-field mismatch | ImageHeader | image_size ≠ actual data length |
| Algorithm spoofing | ImageHeader | algorithm = 0xFF (invalid) |
| Version rollback | ImageHeader | version < current (should be rejected) |
| Signature truncation | SignatureBlock | sig_length = 0 or sig_length > actual |
| URL injection | UpdateInfo | Special characters in download_url |
| Magic overwrite | All structs | Corrupt magic bytes |
| CRC mismatch | SlotMetadata | Valid fields, invalid CRC |
| NULL dereference | All pointers | Zero-length or null buffer inputs |

---

## 6. Expected Behavior (Pass Criteria)

For ALL fuzzed inputs, the system MUST:

| Requirement | Verification |
| --- | --- |
| Never crash (no segfault, no hard fault, no panic) | ASAN/UBSAN clean |
| Never enter undefined state | State machine invariant check |
| Never execute unverified code | Verification-before-execution invariant |
| Always return an error for invalid input | Error code audit |
| Never leak memory (if dynamic allocation) | LeakSanitizer clean |
| Never access out-of-bounds memory | ASAN clean |
| Never have data-dependent timing on secret data | ct-verif clean (for crypto paths) |

---

## 7. Fuzzing Campaign Parameters

| Parameter | Minimum | Recommended |
| --- | --- | --- |
| Duration per target | 24 hours | 7 days |
| Executions per second | 100+ | 1000+ |
| Seed corpus size | 10+ valid inputs | 100+ covering all field combinations |
| Crash triage | All crashes reviewed | Root cause for each unique crash |
| Coverage threshold | 80% line coverage in parser | 95%+ line coverage |

---

## 8. Integration with CI/CD

```
Build → Unit Tests → Fuzz (short: 1h per target) → Static Analysis → Sign
                                                          │
                                          Fuzz (long: 24h per target) → Report
                                          (runs nightly on release branches)
```

- Short fuzz: Every commit to main (1 hour per target, quick regression)
- Long fuzz: Nightly on release branches (24 hours per target, thorough)
- Pre-release fuzz: 7-day campaign on release candidate

---

## 9. Results and Reporting

### 9.1 Fuzzing Report Template

```
Fuzz Campaign Report
====================
Target: [parser name]
Duration: [hours]
Executions: [total]
Coverage: [line% / branch%]

Crashes found: N
  - CRASH-001: [description] → fixed in commit [hash]
  - CRASH-002: [description] → not a bug (assertion firing correctly)

Hangs found: N (all resolved)

Unique coverage paths: N
New paths in last 24h: N (should approach 0 — saturation)
```

### 9.2 Crash Triage Process

```
1. Reproduce: Does it reproduce with the same input?
2. Minimize: Reduce input to minimal crashing case
3. Root cause: Is this a bug in the parser, a false positive, or expected behavior?
4. Fix: If bug → fix, add regression test with crashing input
5. Verify: Re-run fuzzer with fix → crash no longer reproduces
```

---

## 10. References

| Document | Reference |
| --- | --- |
| Data Structures | `../02_System_Design/Data_Structures.md` |
| Image Format | `../02_System_Design/Image_Format.md` |
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| OTA Flow Pseudocode | `../03_Subsystem/Update/OTA_Flow_Pseudocode.md` |
| Interface Contracts | `../03_Subsystem/Interface_Contracts.md` |
| Side-Channel Countermeasures | `../04_Security/Side_Channel_Countermeasures.md` |
| Attack Tree | `../04_Security/Attack_Tree.md` (A1.1.2: Parser exploit) |
