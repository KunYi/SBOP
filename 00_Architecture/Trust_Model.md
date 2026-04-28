# Trust Model

**Document ID:** SEC-TRUST-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the trust model for SBOP: what is trusted, what is not, how trust is established, and how trust propagates through the system. The trust model is the foundation for all security design decisions.

---

## 2. Root of Trust (RoT)

### 2.1 Manufacturing Root of Trust

**KR (Root Key)** — 256-bit symmetric key generated in a FIPS 140-2 Level 3+ HSM.

- Generated during key ceremony with multi-party quorum
- Never leaves HSM (derive-only operations)
- All device trust chains originate from KR
- KR compromise = all devices untrusted (catastrophic)

### 2.2 Device Root of Trust

**KD (Device Key)** — derived as HKDF(KR, UID), unique per device.

- Injected during provisioning into secure element / TEE
- Stored in hardware-isolated storage
- Accessed via KeyRef handles only (never raw export)
- KD is the device's root of trust for all operations

### 2.3 Image Root of Trust

**KI public key** — embedded in bootloader at build time.

- Used to verify firmware signatures
- Trust established at build time (KI public compiled into Zone 1)
- KI private key never leaves HSM

---

## 3. Trust Boundaries

### 3.1 Device Boundary

```
┌─────────── Trust Boundary ───────────┐
│                                       │
│  ┌─────────┐   ┌─────────┐          │
│  │ Zone 1  │   │ Secure  │          │
│  │ (Boot)  │   │ Element │          │
│  │ Trusted │   │ Trusted │          │
│  └────┬────┘   └─────────┘          │
│       │                               │
│  ┌────▼────────────────────────┐     │
│  │ Zone 2 (Application)         │     │
│  │ UNTRUSTED — must not access  │     │
│  │ Zone 1 memory or keys        │     │
│  └─────────────────────────────┘     │
│                                       │
└───────────────────────────────────────┘
```

**Enforcement:** MPU/MMU configured during LOCK_BOOT phase. Zone 1 memory locked read-only. Zone 2 cannot access Zone 1 memory regions.

### 3.2 Network Boundary

All network communication is untrusted. Device must:
- Verify TLS server certificate (no self-signed)
- Authenticate via mutual TLS (device client certificate)
- Independently verify downloaded firmware signature and hash

### 3.3 Backend Boundary

Backend is trusted for:
- Policy decisions (which device gets which version)
- Device identity verification
- Telemetry ingestion and anomaly detection

Backend is NOT trusted for:
- Firmware integrity (device always re-verifies)
- Key material exposure (backend never has raw KD, only HKDF derivation capability)

### 3.4 Manufacturing Boundary

Manufacturing is partially trusted:
- Station trusted during provisioning window only
- Station attestation verified before each session
- KD temporarily on station (zeroized after injection)
- Station network-isolated from internet
- Backend verifies registration before device ships

---

## 4. Trust Chain

```
KR (HSM, air-gapped, quorum)
 │
 ├──→ KD = HKDF(KR, UID)
 │     │  Device identity, unique per device
 │     │  Stored in secure element / TEE
 │     │
 │     ├──→ KD_Auth → Device authentication (mTLS)
 │     ├──→ KD_Debug → Debug authentication
 │     └──→ KD_Storage → Flash encryption (optional)
 │
 └──→ KI (key pair)
       │  Firmware signing key, in HSM
       │
       └──→ KI_public → Bootloader → Firmware verification
```

**Trust rule:** Every operation must chain back to KR. No trust without cryptographic proof.

---

## 5. Trust Rules

| # | Rule |
| --- | --- |
| 1 | No trust without verification — every input, image, or identity must be cryptographically verified |
| 2 | Trust must be explicitly established — no implicit trust across boundaries |
| 3 | Trust is not transitive by default — Zone 2 trusting backend does not mean Zone 1 trusts Zone 2 |
| 4 | Trust can be revoked — keys rotate; compromised devices are removed from backend |
| 5 | Trust is bounded by time — session tokens expire; certificates have validity periods |
| 6 | Debug does not imply trust — even authenticated debug must not expose key material |

---

## 6. What Is NOT Trusted

| Element | Why |
| --- | --- |
| Network (any) | Attacker can intercept, modify, replay |
| Application code (Zone 2) | May contain bugs, may be compromised |
| Downloaded firmware (before verification) | Could be anything — verify first |
| Manufacturing station (outside session window) | Station could be compromised |
| Backend (for firmware integrity) | CDN could be compromised — device re-verifies |
| Debug port (even after auth) | Debug auth could be bypassed — keys still protected |

---

## 7. References

| Document | Reference |
| --- | --- |
| Threat Model | `Threat_Model.md` |
| Security Goals | `../04_Security/Security_Goals.md` |
| Zone Conduit Model | `../04_Security/Zone_Conduit_Model.md` |
| Key Hierarchy | `../02_System_Design/Key_Hierarchy.md` |
| Architecture Decision Record | `Architecture_Decision_Record.md` |
