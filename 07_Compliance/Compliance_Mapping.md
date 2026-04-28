# Compliance Mapping

**Document ID:** CMP-MAP-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Maps SBOP system elements to compliance standards. This is the entry-point document — detailed clause-by-clause mappings are in separate documents per standard.

---

## 2. Standards Summary

| Standard | Domain | SBOP Relevance | Detailed Mapping |
|----------|--------|----------------|------------------|
| ISO/SAE 21434 | Automotive cybersecurity | Full lifecycle (clauses 5-15) | `ISO_21434_Detailed_Mapping.md` |
| IEC 62443 | Industrial control systems | FR 1-7, SL-T/A/C, 62443-4-1 SDLC | `IEC_62443_Detailed_Mapping.md` |
| ECSS-E-ST-40C, Q-ST-80C, E-ST-70-41C | Space engineering | Software, product assurance, telemetry | `ECSS_Compliance_Mapping.md` |
| IEC 62304 / ISO 14971 | Medical devices | Software lifecycle, risk management | `Medical_Compliance_Mapping.md` |
| DO-178C / DO-254 / DO-326A / ARP4754A | Aviation | Software, hardware, security process | `Aviation_Compliance_Mapping.md` |
| IEC 61508 / ISO 26262 | Functional safety | SIL/ASIL, fault injection | `../04_Security/Safety_Analysis.md` |

---

## 3. Domain-to-Subsystem Mapping

| SBOP Domain | ISO 21434 | IEC 62443 | ECSS | IEC 62304 | DO-178C |
|-------------|-----------|-----------|------|-----------|---------|
| Boot (Secure) | §10.4.1 (Security controls) | CR 3.2 (Malicious code protection) | E-ST-40 §5.4.3 (Boot SW) | §5.3 (Architecture) | Level A (MC/DC) |
| OTA Update | §10.4.2 (Update integrity) | CR 5.2 (Integrity verification) | E-ST-70-41 (Telecommand) | §5.7 (Verification) | Level C (Statement) |
| Identity | §9.5 (TARA asset) | CR 6.2 (Authentication) | Q-ST-80 §5.4.2 | §5.5 (Interfaces) | Level B |
| Crypto | §10.4.1 (Cryptographic controls) | CR 4.2 (Least privilege) | Q-ST-80 §5.4.3 | — | Level A |
| Supply Chain | §12 (Production) | 62443-4-1 SM-1..SM-13 | Q-ST-80 §5.3.2 | — | DO-326A |
| Physical | §10.4.3 (Physical) | CR 7.1 (Resource availability) | E-ST-40 §5.4.3 | — | DO-254 |
| Side-Channel | §10.4.1 (Crypto) | — | Q-ST-80 §5.4.3 | — | — |
| Debug | §10.4.1 (Access control) | CR 6.2 | — | §5.3.5 (Segregation) | — |
| Manufacturing | §12 (Production) | — | Q-ST-80 §5.3.2 | — | — |

---

## 4. Requirement-to-Standard Trace

| Requirement | ISO 21434 | IEC 62443 | ECSS | IEC 62304 | DO-178C |
|-------------|-----------|-----------|------|-----------|---------|
| REQ-FR-BOOT-001 | §10.4.1 | CR 3.2 | §5.4.3 | §5.3.5 | §6.4.3 (Level A) |
| REQ-FR-BOOT-003 | §10.4.1 | CR 5.2 | §5.4.3 | §5.7.3 | §6.4.4 |
| REQ-FR-BOOT-004 | §10.4.1 | CR 5.2 | — | — | §6.4.4 |
| REQ-FR-OTA-001 | §10.4.2 | CR 5.2, 6.2 | §5.4.3 | §5.7.3 | — |
| REQ-FR-OTA-003 | §10.4.2 | CR 7.2 | — | §5.7.3 | — |
| REQ-SR-ID-001 | §9.5 (Asset) | CR 6.2 | §5.4.2 (Req) | §5.5.3 | — |
| REQ-SR-SC-001 | §12 | 62443-4-1 SM-1 | §5.3.2 | — | DO-326A §3 |
| REQ-SR-PHY-001 | §10.4.3 | CR 7.1 | — | — | DO-254 |
| REQ-SR-SCH-001 | §10.4.1 | — | §5.4.3 | — | — |
| REQ-SR-DBG-001 | §10.4.1 | CR 6.2 | — | §5.3.5 | — |
| REQ-SR-MFR-001 | §12 | — | §5.3.2 | — | — |
| REQ-NFR-SYS-003 | — | CR 7.2 | §5.4.2 | §5.7.3 | §6.3.3 |

---

## 5. Detailed Standards-Specific Documents

| Standard | Document | Contents |
|----------|----------|----------|
| ISO/SAE 21434 | `ISO_21434_Detailed_Mapping.md` | Clauses 5-15 mapped to SBOP artifacts, TARA evidence, gap analysis |
| IEC 62443 | `IEC_62443_Detailed_Mapping.md` | FR 1-7, SL-T/A/C targets, 62443-4-1 SDLC mapped, gap analysis |
| ECSS (Space) | `ECSS_Compliance_Mapping.md` | E-ST-40, Q-ST-80, E-ST-70-41 clause-by-clause, radiation considerations |
| Medical | `Medical_Compliance_Mapping.md` | IEC 62304 Class C, ISO 14971 risk matrix, FDA 21 CFR 820, EU MDR, SOUP |
| Aviation | `Aviation_Compliance_Mapping.md` | DO-178C Levels A-E, DO-254 hardware assurance, DO-326A security process |
| Zone/Conduit | `../04_Security/Zone_Conduit_Model.md` | 6 zones, 4 conduits, trust matrix per IEC 62443 |
| CAL/SL-T | `../04_Security/CAL_SLT_Definitions.md` | CAL 1-4, SL 1-4 definitions, achievement plan |
| TARA | `../04_Security/TARA_Methodology.md` | ISO 21434 §9.5 compliant TARA: asset inventory, threat scenarios, risk evaluation |
| Safety | `../04_Security/Safety_Analysis.md` | IEC 61508 SIL, ISO 26262 ASIL, HARA hazard analysis |

---

## 6. References

| Document | Reference |
|----------|-----------|
| Compliance Overview | `Compliance_Overview.md` |
| Audit Trace | `Audit_Trace.md` |
| Work Products | `Work_Products.md` |
