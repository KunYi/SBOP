# SBOP Key Hierarchy

**Document ID:** SYS-KH-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the complete logical structure of cryptographic keys, their derivation paths, usage, and protection requirements.

---

## 2. Key Hierarchy Diagram

```
KR (Root Key) — 256-bit symmetric, FIPS 140-2 L3+ HSM
 │  Generated during key ceremony (multi-party quorum)
 │  Never leaves HSM — derive-only operations
 │
 ├──→ KD = HKDF(KR, UID)
 │     │  Device master key, unique per device
 │     │  256-bit, stored in secure element / TEE
 │     │
 │     ├──→ KD_Auth   = HKDF(KD, "SBOP-AUTH-v1")
 │     │     │  Device authentication (mTLS client cert)
 │     │     │  256-bit, used to derive TLS client certificate
 │     │     │
 │     ├──→ KD_Debug  = HKDF(KD, "SBOP-DEBUG-v1")
 │     │     │  Debug port authentication
 │     │     │  256-bit, challenge-response protocol
 │     │     │
 │     └──→ KD_Storage = HKDF(KD, "SBOP-STORAGE-v1")
 │           │  Flash encryption key (optional, platform-dependent)
 │           │  256-bit, used only when flash encryption is enabled
 │           │
 └──→ KI (Image Signing Key) — Ed25519 key pair
       │  Firmware signing, private key in HSM
       │  Public key embedded in bootloader at build time
       │
       └──→ KI_public → Bootloader → Firmware signature verification
```

---

## 3. Key Definitions

| Key | Type | Size | Derivation | Storage | Scope |
|-----|------|------|------------|---------|-------|
| KR | Symmetric | 256-bit | Generated in HSM ceremony | HSM (air-gapped) | Global — all devices |
| KD | Symmetric | 256-bit | HKDF(KR, UID) | Secure element / TEE | Per-device |
| KD_Auth | Symmetric | 256-bit | HKDF(KD, "SBOP-AUTH-v1") | Secure element / TEE | Per-device |
| KD_Debug | Symmetric | 256-bit | HKDF(KD, "SBOP-DEBUG-v1") | Secure element / TEE | Per-device |
| KD_Storage | Symmetric | 256-bit | HKDF(KD, "SBOP-STORAGE-v1") | Secure element / TEE | Per-device (optional) |
| KI (private) | Ed25519 | 256-bit | Generated in HSM | HSM | Per-firmware-version or per-release |
| KI (public) | Ed25519 | 256-bit / 256-bit | — | Bootloader (Zone 1, read-only) | Per-firmware-version |

---

## 4. Key Usage Matrix

| Operation | Key Used | Direction | Context |
|-----------|----------|-----------|---------|
| Device mTLS authentication | KD_Auth | Device → Backend | OTA session establishment |
| Debug port unlock | KD_Debug | Debug Host → Device | Challenge-response protocol |
| Flash encryption (optional) | KD_Storage | Device internal | Transparent flash encryption |
| Firmware signature verification | KI (public) | Device Boot | Every boot |
| Firmware signing | KI (private) | Build pipeline → HSM | Release signing ceremony |
| Attestation proof | KD | Device → Backend | Provisioning registration |
| Session token derivation | KD_Auth | Device + Backend | OTA session |

---

## 5. Key Protection Requirements

| Key | Protection | Access Control |
|-----|------------|----------------|
| KR | FIPS 140-2 L3+ HSM, air-gapped, quorum-based access | Multi-party authorization required |
| KD | Secure element / TEE, KeyRef handles only, never raw export | Zone 1 only (post-boot) |
| KD_Auth | Same as KD | Zone 1 only, used for TLS handshake |
| KD_Debug | Same as KD | Zone 1 only, used for debug challenge-response |
| KD_Storage | Same as KD | Zone 1 only, passed to flash controller |
| KI (private) | HSM, quorum-based signing ceremony | Build pipeline (authorized release engineers) |
| KI (public) | Embedded in bootloader, Zone 1 read-only after LOCK_BOOT | Readable by all zones (public key) |

---

## 6. Key Lifecycle

| Key | Creation | Rotation | Revocation | Zeroization |
|-----|----------|----------|------------|-------------|
| KR | Key ceremony (once) | Catastrophic only (all devices decommissioned) | All devices untrusted | HSM decommission |
| KD | Provisioning (per device) | Device decommission + re-provision | Backend revocation list | Tamper detection → zeroize secure element |
| KD_Auth/KD_Debug/KD_Storage | Derived from KD at first use | Regenerated when KD rotates | Same as KD | Same as KD |
| KI | Key ceremony (per release or version) | Per release (backward compatible: old KI_public retained for previous images) | Revoke specific firmware versions | HSM key deletion |

---

## 7. Derivation Constraints

1. KD derivation requires KR in HSM — only possible during provisioning
2. Sub-keys (KD_Auth, KD_Debug, KD_Storage) derived from KD on-device at first use
3. HKDF info strings are versioned ("SBOP-AUTH-v1") — enables future algorithm migration
4. No key below KR can be derived without: KR (in HSM) + UID, OR KD (on-device)
5. KI is independent of device identity — same KI verifies all devices

---

## 8. References

| Document | Reference |
|----------|-----------|
| Trust Model | `../00_Architecture/Trust_Model.md` |
| Crypto Algorithms | `../03_Subsystem/Crypto/Crypto_Algorithms.md` |
| Key Derivation | `../03_Subsystem/Crypto/Key_Derivation.md` |
| Identity Overview | `../03_Subsystem/Identity/Identity_Overview.md` |
| Provisioning Flow | `../03_Subsystem/Identity/Provisioning_Flow.md` |
