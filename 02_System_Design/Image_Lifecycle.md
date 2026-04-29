# Firmware Image Lifecycle

**Document ID:** SYS-ILC-001
**Version:** 2.1
**Status:** Draft
**Last Review:** 2026-04-29

---

## 1. Purpose

Defines how firmware images are created, transferred, validated, and executed, mapped to the 7-step build pipeline.

---

## 2. Lifecycle Stages (7 Steps)

| Stage | Owner | Action | Output | Verification |
|-------|-------|--------|--------|--------------|
| 1. Build | Build pipeline | Compile source, link, generate binary | Unsigned firmware binary | Reproducible build verification |
| 2. Package | Build pipeline | Compute SHA-256 hash, construct ImageHeader | Packaged (unsigned) image | Hash verification |
| 3. Sign | HSM | Ed25519 sign hash, attach signature | Signed firmware image | Signature self-test |
| 4. Encrypt | Build pipeline + HSM | X25519 ECDH + AES-256-GCM encrypt signed image | OTA Image Package (encrypted) | GCM tag verify + decrypt self-test |
| 5. Distribute | Backend | Store in firmware registry, assign version, publish | Published OTA Image Package | Registry integrity check |
| 6. Download | Device (OTA) | Download via TLS 1.3 mutual auth to buffer | Downloaded OTA package (unverified) | TLS integrity + GCM auth |
| 7. Verify | Device (Boot/OTA) | ECDH decrypt → verify Ed25519 signature + SHA-256 hash + version | Verified plaintext firmware | Full re-verification |
| 8. Execute | Device (Boot) | Re-encrypt with KD_Storage, mark slot ACTIVE, lock Zone 1, jump to entry | Running firmware | Boot state machine guards |

---

## 3. Stage Details

### 3.1 Build (Pipeline)

- Source code committed with GPG-signed commits
- Reproducible build environment (containerized, pinned toolchain)
- SBOM generated (dependencies, toolchain versions, build parameters)
- Build output: unsigned binary + SBOM artifact

### 3.2 Package (Pipeline)

- SHA-256 hash computed over binary
- ImageHeader constructed: magic, version, header_size, image_size, hash_algorithm, hash, sig_algorithm, sig_size, slot, flags
- Packaged image = ImageHeader + ImageBody

### 3.3 Sign (HSM)

- Hash from ImageHeader sent to HSM (air-gapped or network-isolated)
- HSM signs with KI private key (Ed25519)
- Signature appended to image: ImageHeader + ImageBody + SignatureBlock
- Signing ceremony: multi-party authorization, audit logged
- Post-signing: signature self-test (verify with KI public)

### 3.4 Encrypt (Build Pipeline + HSM)

- Input: Signed SBOP image (ImageHeader + Firmware + SignatureBlock)
- Generate ephemeral X25519 key pair (fresh per OTA release)
- Compute ECDH shared secret: `SharedSecret = X25519(Ephemeral_Private, KO_public)`
- Derive session key: `K_s = HKDF-Expand(SharedSecret, "SBOP-OTA-v1" || FirmwareVersion, 32)`
- Derive GCM nonce: `Nonce = HKDF-Expand(SharedSecret, "SBOP-OTA-NONCE-v1", 12)`
- AES-256-GCM encrypt the signed SBOP image with K_s, Nonce, and AAD
- Assemble OTA Image Package: OTAHeader(ephemeral_pubkey) || AAD || EncryptedPayload || GCM_Tag || Ed25519_Signature
- Post-encryption self-test: decrypt with K_s, verify Ed25519 signature on recovered plaintext
- Zeroize ephemeral private key, K_s, and SharedSecret after self-test
- Output: OTA Image Package ready for distribution

**Sign-then-encrypt rationale:** The Ed25519 signature (from stage 3) covers the plaintext firmware. It is stored in the OTA Image Package alongside the ciphertext. The device verifies the signature after GCM decryption. This ordering ensures:
1. GCM authenticates the ciphertext (symmetric integrity — detects transmission errors)
2. Ed25519 authenticates the plaintext (asymmetric authenticity — proves Dev Team origin)
3. Both checks must pass before the firmware is trusted

**Ephemeral key management:** Each OTA release uses a unique ephemeral X25519 key pair. The private key is zeroized immediately after packaging. The public key is embedded in the OTA header for device-side ECDH. This provides forward secrecy per release — compromise of one release's K_s does not compromise other releases.

**AAD binding:** `AAD = firmware_version || device_class || timestamp` — binds the ciphertext to firmware metadata, preventing cross-version or cross-class ciphertext substitution.

→ See `../03_Subsystem/Crypto/Crypto_Algorithms.md` §5-6 for algorithm details.
→ See `../03_Subsystem/Crypto/Key_Derivation.md` §2.1 for K_s derivation.

### 3.5 Distribute (Backend)

- Signed image uploaded to firmware registry
- Metadata assigned: version, target device groups, rollout policy
- Registry maintains integrity (hash chain or signed manifest)
- CDN distributes to devices on request

### 3.6 Download (Device OTA)

- Device authenticates via mTLS (KD_Auth client certificate)
- Backend authorizes: device in group, version applicable, policy allows
- Device downloads to RAM buffer (not directly to flash)
- Range requests supported for interruption recovery
- Download complete: image in buffer, not yet trusted

### 3.7 Verify (Device)

- **Device always re-verifies independently of backend.** Backend compromise cannot affect device.
- Verify signature (Ed25519) against KI public key in Zone 1
- Verify SHA-256 hash over image body
- Check version against OTP counter (anti-rollback)
- All checks passed → image written to inactive slot
- Any check failed → image discarded, error reported

### 3.8 Execute (Device Boot)

- Boot selects slot (see `Boot_Flow_Pseudocode.md`)
- Full verification chain repeated at boot (steps 5-9 of boot flow)
- OTP counter committed if version > current
- Slot marked ACTIVE
- Zone 1 locked (MPU/MMU), jump to Zone 2 entry point

---

## 4. State Transitions

```
UNSIGNED ──→ PACKAGED ──→ SIGNED ──→ ENCRYPTED ──→ DISTRIBUTED ──→ DOWNLOADED ──→ VERIFIED ──→ ACTIVE
                                                                                             │
                                                                                             └──→ REJECTED (verification failure)
```

---

## 5. Failure Handling

| Stage | Failure | Response |
|-------|---------|----------|
| Build | Compilation error | Reject — fix source |
| Package | Hash computation error | Reject — check toolchain |
| Sign | HSM unavailable | Block release — retry with quorum |
| Encrypt | ECDH or GCM failure | Block release — check HSM and key material |
| Distribute | Registry corruption | Detect via integrity check — restore from backup |
| Download | Interrupted transfer | Resume via Range request |
| Verify | Signature/hash/version failure | Discard image — re-download or wait for next release |
| Execute | Boot verification failure | FAILSAFE — boot from other slot |

---

## 6. Immutability Principle

- Once signed, the image must never be modified
- Any modification (even one bit) invalidates the signature and hash
- The ImageHeader hash field locks the entire image body
- Version number is part of signed metadata — cannot be changed without re-signing

---

## 7. References

| Document | Reference |
|----------|-----------|
| Image Format | `Image_Format.md` |
| System Flow | `System_Flow.md` |
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| OTA Flow Pseudocode | `../03_Subsystem/Update/OTA_Flow_Pseudocode.md` |
| Supply Chain Security | `../04_Security/Supply_Chain_Security.md` |
| Crypto Algorithms | `../03_Subsystem/Crypto/Crypto_Algorithms.md` |
