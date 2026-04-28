# Threat Model

**Document ID:** SEC-TM-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the threat actors, their capabilities, motivations, and attack goals relevant to SBOP. This threat model is the foundation for the attack tree, risk assessment, and security control design.

→ Outputs: `Attack_Tree.md`, `TARA_Methodology.md`, `Risk_Quantification.md`

---

## 2. Threat Actors

### 2.1 Actor Profiles

| Actor | Capability Level | Motivation | Typical Resources | Targets |
| --- | --- | --- | --- | --- |
| **Script Kiddie** | Low — uses public tools only | Curiosity, notoriety | $0 budget, days of effort | Known vulnerabilities, default configs |
| **Skilled Hobbyist** | Medium — custom scripts, basic HW | Challenge, reputation | $1K budget, weeks of effort | Implementation bugs, weak crypto |
| **Organized Crime** | High — professional tools, dedicated team | Financial gain | $100K budget, months of effort | Device cloning, ransomware, IP theft |
| **Competitor** | High — domain expertise, insider knowledge | Market advantage | $500K budget, months of effort | IP extraction, cloning, sabotage |
| **Nation-State** | Very High — custom silicon, FIB, lab facilities | Espionage, sabotage, military | $1M+ budget, years of effort | Full compromise, supply chain, backdoors |
| **Insider (Engineer)** | Medium-High — access + knowledge | Grievance, financial, coercion | Internal access, months of opportunity | Key theft, malicious commit, backdoor |
| **Insider (Operator)** | Medium — operational access | Grievance, financial | Physical + logical access | Unauthorized signing, provisioning fraud |

### 2.2 Capability Mapping to Security Levels

| Capability | SL 1 | SL 2 | SL 3 | SL 4 |
| --- | --- | --- | --- | --- |
| Network attacks (MITM, replay) | ✓ | ✓ | ✓ | ✓ |
| Software exploitation (parsing bugs) | — | ✓ | ✓ | ✓ |
| Hardware debugging (JTAG/SWD) | — | ✓ | ✓ | ✓ |
| Side-channel (timing) | — | — | ✓ | ✓ |
| Fault injection (voltage/clock glitch) | — | — | ✓ | ✓ |
| Side-channel (power/EM DPA) | — | — | ✓ | ✓ |
| Chip decapsulation | — | — | — | ✓ |
| Microprobing / FIB | — | — | — | ✓ |
| Supply chain interdiction | — | — | — | ✓ |

→ See `CAL_SLT_Definitions.md` for detailed SL capability ratings.

---

## 3. Attack Goals

### 3.1 Primary Goals

| Goal | Impact | Motivation |
| --- | --- | --- |
| Execute unauthorized firmware | Complete device compromise | All actors |
| Extract cryptographic keys | Compromise fleet, sign malware | Organized crime, nation-state |
| Clone device identity | Counterfeiting, unauthorized access | Organized crime, competitor |
| Disable or degrade device | Denial of service, extortion | Nation-state, organized crime |
| Exfiltrate sensitive data | Privacy breach, IP theft | All actors |
| Establish persistent backdoor | Long-term access, espionage | Nation-state |

### 3.2 Secondary Goals

| Goal | Impact | Motivation |
| --- | --- | --- |
| Bypass OTA update mechanism | Prevent security patches | Nation-state, competitor |
| Inject false telemetry | Hide compromise, mislead operations | Nation-state, organized crime |
| Overproduce devices | Revenue loss, gray market | Insider (operator), competitor |
| Compromise supply chain | Compromise at scale | Nation-state |

---

## 4. Attack Surfaces

### 4.1 External Attack Surfaces

| Surface | Accessibility | Exploitability | Impact if Compromised |
| --- | --- | --- | --- |
| OTA download (TLS endpoint) | Remote (network) | Low (TLS + signature + hash) | Fleet-wide firmware compromise |
| Telemetry upload (TLS endpoint) | Remote (network) | Low (TLS + auth) | Information disclosure |
| Device network interface | Remote (network) | Depends on application | Application compromise → Zone 2 |

### 4.2 Local Attack Surfaces

| Surface | Accessibility | Exploitability | Impact if Compromised |
| --- | --- | --- | --- |
| Debug port (JTAG/SWD) | Physical | Low (locked) / Medium (if unlocked) | Full device compromise |
| Flash memory (external) | Physical | Medium (encrypted / chip-off) | Firmware extraction, cloning |
| Power/clock lines | Physical | Medium (fault injection) | Verification bypass |
| Tamper switches | Physical | Low (detected) | Disabled tamper detection |
| UART/serial console | Physical | Medium (if enabled) | Debug information leakage |

### 4.3 Supply Chain Attack Surfaces

| Surface | Accessibility | Exploitability | Impact if Compromised |
| --- | --- | --- | --- |
| Source code repository | Logical (network) | Medium (auth + review) | All devices running compromised code |
| Build server | Logical (network) | Medium (isolated) | Compromised firmware images |
| Signing HSM | Logical + physical | Very Low (quorum + air gap) | All devices compromised |
| CDN / firmware distribution | Logical (network) | Low (TLS + device-side verification) | Fleet compromise if device checks bypassed |
| Provisioning station | Physical + logical | Medium (isolated + audited) | Per-batch identity compromise |

---

## 5. Trust Boundaries

### 5.1 Trust Boundary Diagram

```
┌──────────────────────────────────────────────────────┐
│                   Untrusted Zone                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Internet │  │ Physical │  │ Supply Chain     │   │
│  │ Attacker │  │ Attacker │  │ Attacker         │   │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
└───────┼──────────────┼─────────────────┼─────────────┘
        │              │                 │
   ═════╪══════════════╪═════════════════╪═══════════ Trust Boundary 1: Network/Physical
        │              │                 │
┌───────┼──────────────┼─────────────────┼─────────────┐
│       ▼              ▼                 ▼              │
│  ┌─────────┐   ┌──────────┐   ┌──────────────┐      │
│  │ Backend │   │ Device   │   │ Build & Sign │      │
│  │ Server  │   │ Enclosure│   │ Pipeline     │      │
│  └────┬────┘   └────┬─────┘   └──────┬───────┘      │
│       │              │                 │              │
│  ═════╪══════════════╪═════════════════╪══════════ Trust Boundary 2: Application/Zone 2
│       │              │                 │              │
│  ┌────▼────┐   ┌─────▼──────┐         │              │
│  │ App     │   │ Zone 2     │         │              │
│  │ Backend │   │ (OTA, App) │         │              │
│  └────┬────┘   └─────┬──────┘         │              │
│       │              │                 │              │
│  ═════╪══════════════╪═════════════════╪══════════ Trust Boundary 3: Zone 1 (Critical)
│       │              │                                │
│  ┌────▼────┐   ┌─────▼──────┐                        │
│  │ Key     │   │ Zone 1     │                        │
│  │ Storage │   │ (Boot,     │                        │
│  │ (HSM)   │   │  Crypto,   │                        │
│  │         │   │  Identity) │                        │
│  └─────────┘   └────────────┘                        │
│                                                       │
│                  Trusted Execution Base               │
└──────────────────────────────────────────────────────┘
```

### 5.2 Trust Boundary Descriptions

| Boundary | What It Separates | Enforcement |
| --- | --- | --- |
| TB1: Network/Physical | External world from SBOP infrastructure | TLS 1.3, physical security, network isolation |
| TB2: Application/Zone 2 | Application from secure core | MPU/MMU, Zone 1 read-only after boot |
| TB3: Zone 1 (Critical) | Boot/crypto/identity from everything else | Secure element / TEE, no external access to keys |

---

## 6. Attacker Assumptions

### 6.1 Assumed Capabilities

| ID | Assumption | Applies To |
| --- | --- | --- |
| ASM-ATT-01 | Attacker controls the network (can intercept, modify, replay) | All SL |
| ASM-ATT-02 | Attacker has physical access to device (can open enclosure) | SL 2+ |
| ASM-ATT-03 | Attacker can control power supply (glitch, power cycle) | SL 3+ |
| ASM-ATT-04 | Attacker has access to debug port (JTAG/SWD) | SL 2+ |
| ASM-ATT-05 | Attacker can make unlimited authentication attempts | All SL (rate-limited by design) |
| ASM-ATT-06 | Attacker has full documentation and source code | All SL (security by design, not obscurity) |

### 6.2 Assumed Non-Capabilities (Design Assumptions)

| ID | Assumption | Rationale |
| --- | --- | --- |
| ASM-DEF-01 | Attacker cannot break ECDSA P-256 / Ed25519 | Cryptographic assumption |
| ASM-DEF-02 | Attacker cannot find SHA-256 collisions | Cryptographic assumption |
| ASM-DEF-03 | Attacker cannot forge TRNG output | Hardware entropy assumption |
| ASM-DEF-04 | FIPS 140-2 L3 HSM prevents key extraction | HSM certification |
| ASM-DEF-05 | Secure element / TEE provides key isolation | Hardware security assumption |

---

## 7. Threat Scenario Catalog

| ID | Scenario | Actor | Goal | Attack Tree |
| --- | --- | --- | --- | --- |
| TS-01 | Remote attacker serves malicious OTA update | Skilled Hobbyist+ | Execute unauthorized firmware | A1.3, A5 |
| TS-02 | Physical attacker glitches verification check | Professional Team+ | Skip signature verification | A1.1.1, A7.2 |
| TS-03 | Attacker extracts keys via side-channel | Professional Team+ | Sign arbitrary firmware | A8.1, A8.2 |
| TS-04 | Insider inserts backdoor in source code | Insider (Engineer) | Persistent access | A6.1.1 |
| TS-05 | Attacker clones device identity | Organized Crime | Unauthorized network access | A4, A10.2 |
| TS-06 | Attacker performs rollback to vulnerable version | Skilled Hobbyist+ | Exploit known vulnerability | A3 |
| TS-07 | Attacker compromises build server | Nation-State | Compromise all firmware | A6.2.1 |
| TS-08 | Attacker brute-forces debug authentication | Skilled Hobbyist+ | Full device access | A9.2.2 |
| TS-09 | Rogue factory overproduces devices | Insider (Operator) | Unauthorized devices on network | A10.1.1 |
| TS-10 | Zone 2 compromise escalates to Zone 1 | Skilled Hobbyist+ | Bypass boot verification | A11.1 |

---

## 8. Threat Model Maintenance

### 8.1 Update Triggers

- New attack technique demonstrated (e.g., new fault injection method)
- New vulnerability class affecting embedded systems
- Change in attacker capability landscape
- New deployment context with new threat actors
- Post-incident review findings

### 8.2 Review Cycle

| Activity | Frequency |
| --- | --- |
| Threat model review | Annual + per major release |
| Attack tree update | When new attacks identified |
| TARA re-assessment | When threat actors or capabilities change |
| Assumptions review | Annual (verify cryptographic and hardware assumptions still hold) |

---

## 9. References

| Document | Reference |
| --- | --- |
| Attack Tree | `../04_Security/Attack_Tree.md` |
| TARA Methodology | `../04_Security/TARA_Methodology.md` |
| Risk Quantification | `../04_Security/Risk_Quantification.md` |
| CAL/SL-T Definitions | `../04_Security/CAL_SLT_Definitions.md` |
| Zone Conduit Model | `../04_Security/Zone_Conduit_Model.md` |
| Security Goals | `../04_Security/Security_Goals.md` |
