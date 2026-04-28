# SBOP System Context

**Document ID:** ARC-CTX-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Overview

Defines the SBOP system context: external entities, data flows, trust zones, and boundary conditions. This is the top-level context diagram for the entire system.

---

## 2. External Entities

### 2.1 Device

The embedded target running SBOP firmware. Comprises Zone 1 (bootloader, crypto, identity) and Zone 2 (application, OTA client). The device is the primary protected asset.

### 2.2 Backend Service

Cloud or on-premise infrastructure providing:
- Firmware registry (signed images, metadata)
- Device registry (identities, groups, status)
- Key management (KR storage, KD derivation, KI management)
- Deployment controller (staged rollout, abort, rollback)
- Telemetry ingestion and anomaly detection

### 2.3 Manufacturing System

Provisioning stations in factories that:
- Generate or inject device identity (UID, KD)
- Register devices with backend
- Perform self-tests and lock devices
- Operate in isolated network zones

### 2.4 Developer / Operator

- Firmware developers: write source code, commit to repository
- Release engineers: run build pipeline, trigger signing ceremony
- Security architect: reviews design, accepts risks, manages keys
- Operations: monitors fleet, responds to incidents

### 2.5 Attacker (External Entity)

Always present in the threat model. See `Threat_Model.md` for profiles and capabilities.

---

## 3. Context Diagram

```
                        ┌──────────────────────┐
                        │     Backend Service   │
                        │  ┌────────────────┐   │
                        │  │ Firmware       │   │
                        │  │ Registry       │   │
                        │  │ Device         │   │
                        │  │ Registry       │   │
                        │  │ Key Manager    │   │
                        │  │ Deploy Ctrl    │   │
                        │  └───────┬────────┘   │
                        └──────────┼────────────┘
                                   │
                    TLS 1.3 + Mutual Auth
                                   │
    ┌──────────┐        ┌──────────┴──────────┐
    │ Attacker │ ─ ─ ─ │      Device           │
    └──────────┘        │  ┌────────────────┐   │
                        │  │ Zone 1: Boot   │   │
    ┌──────────┐        │  │ + Crypto + ID  │   │
    │ Developer│        │  ├────────────────┤   │
    │ / Ops    │        │  │ Zone 2: App    │   │
    └────┬─────┘        │  │ + OTA Client   │   │
         │              │  └────────────────┘   │
    ┌────▼──────────┐   └──────────────────────┘
    │ Build & Sign  │
    │ Pipeline      │            │
    │ (HSM)         │            │ Provisioning
    └───────────────┘   ┌────────▼──────────┐
                        │  Manufacturing    │
                        │  Station          │
                        └──────────────────┘
```

---

## 4. Data Flows

### 4.1 Firmware Delivery

```
Developer → Source Repo → Build Pipeline → sbop-sign (HSM) → Firmware Registry → Backend → [TLS 1.3] → Device (inactive slot)
```

**Protection:** Commit signing (repo), reproducible build (pipeline), HSM signing, TLS + device-side signature + hash verification.

### 4.2 Device Authentication (OTA)

```
Device → [KD_Auth client cert] → Backend (mutual TLS handshake) → Backend verifies device identity → issues session token
```

**Protection:** TLS 1.3 mutual auth, device certificate derived from KD_Auth, session tokens time-limited.

### 4.3 Key Provisioning

```
Device → [UID from TRNG] → Station → [KR from HSM] → HKDF(KR, UID) → KD → inject to device secure element
```

**Protection:** Encrypted channel (station-to-device), station attestation, backend registration, KD never stored on station after injection.

### 4.4 Telemetry & Status

```
Device → [telemetry record, TLS 1.3] → Backend telemetry ingestion → anomaly detection → dashboard
```

**Protection:** TLS 1.3, device identity hashed (anonymized), HMAC integrity on telemetry records.

---

## 5. Trust Zones

### 5.1 Device Zone

- Executes secure boot and OTA
- Stores KD and derived keys in secure element / TEE
- Zone 1 (boot) is trusted; Zone 2 (application) is untrusted
- MPU/MMU enforces Zone 1/2 isolation

### 5.2 Backend Zone

- Issues firmware and manages keys
- Authenticates devices via mutual TLS
- Enforces deployment policy and device lifecycle
- Trusted but assumed potentially compromised (device-side verification as defense-in-depth)

### 5.3 Manufacturing Zone

- Injects or derives device keys
- Physically secured; network isolated (no internet)
- Station attestation verified before each session
- Temporary trust — keys zeroized from station after provisioning

→ See `../04_Security/Zone_Conduit_Model.md` for the complete 6-zone model.

---

## 6. Boundary Conditions

| Boundary | Condition |
| --- | --- |
| Network | All communication channels are untrusted. TLS 1.3 with mutual auth required. |
| Physical | Devices may operate in hostile environments. Tamper detection active. |
| Backend | Backend is trusted for policy but NOT for data integrity (device always re-verifies). |
| Manufacturing | Factory network is isolated. Station attestation required. |
| Supply Chain | Source, build, and distribution channels are untrusted. Verification at every handoff. |

---

## 7. References

| Document | Reference |
| --- | --- |
| Threat Model | `Threat_Model.md` |
| Trust Model | `Trust_Model.md` |
| System Overview | `System_Overview.md` |
| Zone Conduit Model | `../04_Security/Zone_Conduit_Model.md` |
| Reference Architecture | `../08_Product/Reference_Architecture.md` |
