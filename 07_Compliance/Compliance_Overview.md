# SBOP Compliance Overview

**Document ID:** CMP-OV-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the compliance framework ensuring SBOP meets industry security standards.

---

## 2. Target Standards

### Automotive
* **ISO/SAE 21434** — Road vehicles -- Cybersecurity engineering
  → Detailed mapping: `ISO_21434_Detailed_Mapping.md`

### Industrial
* **IEC 62443** — Industrial communication networks -- Network and system security
  → Detailed mapping: `IEC_62443_Detailed_Mapping.md`

### Space
* **ECSS-E-ST-40C** — Space Engineering -- Software
* **ECSS-Q-ST-80C** — Space Product Assurance -- Software Product Assurance
* **ECSS-E-ST-70-41C** — Telemetry and Telecommand Packet Utilization
  → Detailed mapping: `ECSS_Compliance_Mapping.md`

### Medical
* **IEC 62304** — Medical Device Software -- Software Life Cycle Processes
* **ISO 14971** — Medical Devices -- Application of Risk Management
* **ISO 13485** — Medical Devices -- Quality Management Systems
  → Detailed mapping: `Medical_Compliance_Mapping.md`

### Aviation
* **DO-178C** — Software Considerations in Airborne Systems and Equipment Certification
* **DO-254** — Design Assurance Guidance for Airborne Electronic Hardware
* **DO-326A** — Airworthiness Security Process Specification
* **ARP4754A** — Guidelines for Development of Civil Aircraft and Systems
  → Detailed mapping: `Aviation_Compliance_Mapping.md`

### Functional Safety (Cross-Cutting)
* **IEC 61508** — Functional Safety of E/E/PE Systems
* **ISO 26262** — Road vehicles -- Functional Safety
  → For SBOP, functional safety requirements are integrated into the security architecture via fail-safe design

---

## 3. Compliance Strategy

* Align requirements with standard clauses
* Generate verifiable evidence
* Maintain traceability across lifecycle

---

## 4. Compliance Scope

* Secure boot
* OTA update
* Identity management
* Cryptographic operations
* Supply chain security
* Manufacturing security
* Physical tamper resistance
* Side-channel resistance
* Secure debug

### 4.1 Standards Version Tracking

| Standard | Version | SBOP Mapping Version | Last Review |
| --- | --- | --- | --- |
| ISO/SAE 21434 | 2021 | CMP-ISO21434-001 v1.0 | 2026-04-28 |
| IEC 62443 | 2015+ | CMP-IEC62443-001 v1.0 | 2026-04-28 |
| ECSS-E-ST-40C | 2009 | CMP-ECSS-001 v1.0 | 2026-04-28 |
| IEC 62304 | 2006+A1:2015 | CMP-MED-001 v1.0 | 2026-04-28 |
| ISO 14971 | 2019 | CMP-MED-001 v1.0 (Section 4) | 2026-04-28 |
| DO-178C | 2011 | CMP-AV-001 v1.0 | 2026-04-28 |
| DO-254 | 2000 | CMP-AV-001 v1.0 | 2026-04-28 |
| DO-326A | 2014 | CMP-AV-001 v1.0 | 2026-04-28 |
| ARP4754A | 2010 | CMP-AV-001 v1.0 | 2026-04-28 |
| IEC 61508 | 2010 | CMP-FS-001 v1.0 | 2026-04-28 |
| ISO 26262 | 2018 | CMP-FS-001 v1.0 | 2026-04-28 |

---

## 5. Deliverables

* Work products
* Evidence records
* Audit trace matrix
