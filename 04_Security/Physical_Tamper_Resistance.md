# Physical Tamper Resistance

**Document ID:** SEC-PHY-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the architectural requirements for physical tamper resistance in SBOP deployments. It addresses physical attack vectors and the countermeasures that the system must support at the architectural level.

**Note:** This document specifies architectural requirements and interfaces. Specific hardware implementations (silicon-level tamper response, specific sensor types) are out of Tier-1 scope and are defined by the platform integration layer.

---

## 2. Physical Threat Model

### 2.1 Attacker Profiles

| Profile | Capability | Access Level | Example |
| --- | --- | --- | --- |
| Casual | Simple tools (screwdriver, multimeter) | Device exterior | Curious user, technician |
| Determined | Soldering equipment, logic analyzer | PCB-level access | Hardware hacker, competitor |
| Professional | Microscopy, focused ion beam (FIB), microprobing | Chip-level access | Well-funded adversary |
| Advanced | State-level resources, custom equipment, decapsulation | Silicon-level access | Nation-state actor |

### 2.2 Physical Attack Vectors

| Attack ID | Attack | Target | Difficulty |
| --- | --- | --- | --- |
| PHY-A-001 | PCB probing / bus sniffing | Memory bus, SPI flash | Low-Medium |
| PHY-A-002 | JTAG/SWD exploitation | Debug port | Low |
| PHY-A-003 | SPI flash readout (chip-off) | Firmware storage | Medium |
| PHY-A-004 | Voltage glitch injection | Verification logic | Medium-High |
| PHY-A-005 | Clock glitch injection | State machine | Medium-High |
| PHY-A-006 | EM fault injection | Crypto operations | High |
| PHY-A-007 | Temperature fault injection | System operation | Medium |
| PHY-A-008 | Laser fault injection | Specific logic gates | Very High |
| PHY-A-009 | Package removal / decapsulation | Key storage | High |
| PHY-A-010 | Microprobing | Internal buses | Very High |
| PHY-A-011 | Power analysis (SPA/DPA) | Crypto operations | Medium-High |
| PHY-A-012 | Cold boot attack | RAM contents | Low (if RAM unencrypted) |

---

## 3. Physical Security Architecture

### 3.1 Protection Layers

```
┌─────────────────────────────────────────────┐
│ Layer 4: Active Tamper Response              │
│  - Key zeroization on tamper detect          │
│  - Device lockdown / brick                   │
├─────────────────────────────────────────────┤
│ Layer 3: Tamper Detection                    │
│  - Enclosure tamper switches                 │
│  - Environmental sensors (V, T, clock)       │
│  - Active shield / mesh continuity           │
├─────────────────────────────────────────────┤
│ Layer 2: Physical Obfuscation                │
│  - Epoxy / potting compound                  │
│  - PCB trace hiding (buried vias)            │
│  - Chip-scale package (CSP/BGA)              │
├─────────────────────────────────────────────┤
│ Layer 1: Logical Protection                  │
│  - Secure debug (JTAG lock)                  │
│  - Memory encryption / scrambling            │
│  - Secure element / TEE for keys             │
├─────────────────────────────────────────────┤
│ Layer 0: Platform Foundation                 │
│  - MCU with hardware security features       │
│  - Immutable boot ROM                        │
│  - Hardware unique key (HUK) or PUF          │
└─────────────────────────────────────────────┘
```

SBOP's architecture interacts with Layer 0 (assuming its existence) and directly specifies Layers 1 and 3. Layers 2 and 4 are platform-dependent extensions.

---

## 4. Tamper Detection

### 4.1 Tamper Events

| Event ID | Event | Sensor Type | SBOP Action |
| --- | --- | --- | --- |
| TMP-001 | Enclosure open | Mechanical switch / Hall effect sensor | Enter tamper state; zeroize device keys |
| TMP-002 | Voltage anomaly (glitch detected) | Voltage monitor comparator | Increment glitch counter; fail-safe if threshold exceeded |
| TMP-003 | Clock anomaly (glitch detected) | Clock monitor (RC watchdog) | Increment glitch counter; fail-safe if threshold exceeded |
| TMP-004 | Temperature excursion | Temperature sensor | Throttle / suspend sensitive operations |
| TMP-005 | Active shield breach | Mesh continuity monitor | Immediate key zeroization |
| TMP-006 | Unexpected debug port activity | JTAG/SWD access monitor | Lock debug port; log event |
| TMP-007 | External memory access anomaly | Memory bus integrity monitor | Fail-safe with tamper indication |

### 4.2 Tamper Detection Interface

SBOP defines an abstract tamper detection interface:

```
tamper_detect_init() → Result<(), HW_Error>
tamper_detect_register_handler(event: TamperEvent, handler: fn()) → Result<HandlerID, HW_Error>
tamper_detect_get_state() → TamperState

enum TamperState:
    NORMAL       // No tamper detected
    SUSPICIOUS   // Anomaly threshold approaching
    TAMPERED     // Tamper confirmed; take action
    COMPROMISED  // Device may be compromised; full lockdown
```

---

## 5. Tamper Response

### 5.1 Response Levels

| Level | Trigger | Response | Recovery |
| --- | --- | --- | --- |
| Level 0: Monitor | Anomaly detected | Log event; increment counter | Automatic |
| Level 1: Alert | Anomaly threshold exceeded | Report to backend; restrict functions | Backend authorization |
| Level 2: Lockdown | Tamper confirmed | Enter FAILSAFE; disable boot | OTA recovery via backend |
| Level 3: Destroy | Critical tamper detected | Zeroize keys; brick device | Physical replacement |

### 5.2 Tamper Response Actions

| Action | Description | Reversible? |
| --- | --- | --- |
| Key zeroization | Overwrite all key material (KR, KD) with zeros/random | No (requires re-provisioning) |
| Device lock | Disable all outputs; enter FAILSAFE | Yes (via authenticated recovery) |
| Debug port disable | Blow debug fuse or set permanent lock bit | No (physical only) |
| Permanent brick | Set unrecoverable lock flag in OTP/fuses | No |
| Backend notification | Queue tamper event for next telemetry report | — |

### 5.3 Response Logic (Boot-Level)

During boot, SBOP must:

1. Sample tamper sensors at RESET
2. If TAMPERED or COMPROMISED state detected:
   a. Zeroize KD
   b. Enter FAILSAFE
   c. Signal tamper status externally
3. If NORMAL: proceed with verification flow

---

## 6. Secure Enclosure Requirements

### 6.1 Physical Enclosure

| Requirement | Description |
| --- | --- |
| PHY-ENC-001 | Enclosure must have at least one tamper-evident seal or switch |
| PHY-ENC-002 | Enclosure opening must trigger tamper event within the tamper detection latency budget |
| PHY-ENC-003 | Tamper switch must be non-bypassable from exterior without leaving evidence |
| PHY-ENC-004 | For SL 3 deployments: Tamper switch must be backed by battery or capacitance (active even when device unpowered) |

### 6.2 Potting / Epoxy

| Requirement | Description |
| --- | --- |
| PHY-POT-001 | Critical components (MCU, secure element, key storage) should be covered by opaque potting compound |
| PHY-POT-002 | Potting must resist common solvents (for SL 3+) |
| PHY-POT-003 | Potting removal must risk component destruction (deterrent, not prevention) |

---

## 7. Platform Integration Requirements

### 7.1 Secure Element / TEE Interface

SBOP must interface with a hardware root of trust. Abstract requirements:

| Requirement | Description |
| --- | --- |
| PHY-HW-001 | Platform must provide secure key storage (cannot be read via debug port) |
| PHY-HW-002 | Platform must support memory protection between Zone 1 and Zone 2 |
| PHY-HW-003 | Platform should support memory encryption for external RAM (SL 3+) |
| PHY-HW-004 | Platform should provide physically unclonable function (PUF) or hardware unique key (HUK) for identity binding |
| PHY-HW-005 | Platform must provide tamper-evident storage for monotonic version counter |

### 7.2 Glitch Protection Requirements

| Requirement | Description |
| --- | --- |
| PHY-GLT-001 | Critical comparison operations (signature check, version comparison) must use redundant execution with comparison |
| PHY-GLT-002 | State machine transitions must include parity or CRC check on state variable |
| PHY-GLT-003 | Voltage monitor must trigger tamper response within glitch detection window (configurable; target < 1μs for SL 3+) |
| PHY-GLT-004 | Clock monitor must detect frequency deviation beyond ±10% of nominal |

---

## 8. Physical Security Zones

Physical security zones map to the IEC 62443 logical zones:

| Physical Zone | Maps To | Physical Protection Level |
| --- | --- | --- |
| PZ-1: MCU die | Zone 1 (Device Core) | Highest — chip-level protection |
| PZ-2: PCB (internal) | Zone 1 + Zone 2 | Moderate — enclosure + potting |
| PZ-3: External connectors | Zone 5 (Network) | Minimal — exposed by design |
| PZ-4: Manufacturing station | Zone 4 (Manufacturing) | Controlled facility |

---

## 9. Physical Penetration Testing

### 9.1 Test Methodology

| Test | Target | Success Criteria |
| --- | --- | --- |
| JTAG/SWD probe attempt | Debug port | Connection refused; device remains locked |
| SPI flash dump | External flash | Encrypted/scrambled data recovered (if applicable) |
| Voltage glitch on verification | Signature verification | FAILSAFE entered; no execution |
| Clock glitch on state machine | State transition | FAILSAFE entered; state integrity maintained |
| Enclosure open | Tamper switch | Tamper event recorded; keys zeroized |
| Temperature cycling | Full system | No undefined behavior; operation within spec |

### 9.2 Physical Attack Lab Requirements

- Oscilloscope (≥ 1 GHz bandwidth for glitch characterization)
- Logic analyzer
- Controlled voltage/clock source (glitch generator)
- Thermal chamber (-40°C to +125°C)
- EM probe (for side-channel; → `Side_Channel_Countermeasures.md`)

---

## 10. Requirements Trace

| Physical Attack | SBOP Requirement | Mechanism | Test |
| --- | --- | --- | --- |
| PHY-A-001 (Bus probing) | REQ-SR-PHY-001 | Memory encryption, secure element | Physical pentest |
| PHY-A-002 (JTAG/SWD) | REQ-SR-DBG-001 | Debug disable/lock | Functional + physical test |
| PHY-A-003 (Flash readout) | REQ-SR-PHY-001 | Storage encryption | Physical pentest |
| PHY-A-004 (Voltage glitch) | REQ-SR-PHY-001 | Glitch detection + redundant checks | FI-001, FI-004 |
| PHY-A-005 (Clock glitch) | REQ-SR-PHY-001 | Clock monitor | FI-001 |
| PHY-A-006 (EM fault injection) | REQ-SR-PHY-001 | Shielding + redundant checks | Physical pentest (SL 3+) |
| PHY-A-009 (Decapsulation) | REQ-SR-KEY-001 | Key in secure element | Physical pentest (SL 4) |
| PHY-A-011 (Power analysis) | REQ-SR-SCH-001 | Constant-time crypto | → `Side_Channel_Countermeasures.md` |

---

## 11. Residual Risk

Per the SBOP residual risk policy (→ `Residual_Risk.md`), the following physical attack risks are accepted:

| Attack | Residual Risk | Justification |
| --- | --- | --- |
| Advanced decapsulation (FIB) | Accepted for SL ≤ 3 | Requires nation-state resources; mitigated at SL 4 only |
| Laser fault injection | Accepted for SL ≤ 2 | Requires specialized equipment; monitored at SL 3+ |
| Physical destruction | Accepted (all SL) | Tamper detection + key zeroization limit data exposure |
