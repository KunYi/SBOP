# Supply Chain Security

**Document ID:** SEC-SC-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

This document defines the security requirements and controls for the SBOP firmware supply chain, from source code through build, signing, distribution, and deployment to devices.

---

## 2. Supply Chain Threat Model

### 2.1 Threat Actors

| Actor | Motivation | Capability |
| --- | --- | --- |
| Insider (developer/operator) | Malicious intent or negligence | Access to source, build system, or signing keys |
| External attacker | Financial, espionage, sabotage | May compromise build infrastructure |
| Dependency attacker | Supply chain poisoning | May inject malicious code into dependencies |
| Nation-state attacker | Strategic, espionage | Advanced persistent threat; long-term infiltration |

### 2.2 Threat Scenarios

| Threat ID | Scenario | Impact |
| --- | --- | --- |
| SC-T-001 | Compromised source code (malicious commit) | All firmware from that commit is backdoored |
| SC-T-002 | Compromised build environment | All builds produced on that environment are suspect |
| SC-T-003 | Compromised dependency (package, library) | Malicious code injected via trusted dependency |
| SC-T-004 | Compromised signing key | Attacker can sign arbitrary firmware |
| SC-T-005 | Compromised distribution channel | Attacker replaces firmware during distribution |
| SC-T-006 | Insider threat at manufacturing | Unauthorized firmware loaded during provisioning |
| SC-T-007 | Counterfeit hardware components | Device built with components that bypass security |
| SC-T-008 | Unauthorized firmware build | Unsigned or unauthorized build deployed to devices |

---

## 3. Toolchain Security

### 3.1 Build Environment

| Requirement | Description | Verification |
| --- | --- | --- |
| SC-BLD-001 | Builds must be reproducible (deterministic output) | Compare build artifacts from identical inputs |
| SC-BLD-002 | Build environment must be ephemeral (containerized/isolated) | Audit build environment provisioning |
| SC-BLD-003 | Build logs must be immutable and retained | Log integrity verification |
| SC-BLD-004 | Build must run in offline or air-gapped environment where signing keys exist | Environment audit |
| SC-BLD-005 | All build inputs (source, dependencies, toolchains) must be version-pinned | Version lockfile verification |

### 3.2 Source Code Integrity

| Requirement | Description | Verification |
| --- | --- | --- |
| SC-SRC-001 | All commits must be cryptographically signed (GPG/SSH signing) | Verify signatures in git history |
| SC-SRC-002 | Protected branches require review approval before merge | Branch protection audit |
| SC-SRC-003 | Two-person rule for merges to release branches | Audit merge approval records |
| SC-SRC-004 | Source provenance must be traceable from binary to commit | Build provenance metadata |

### 3.3 Dependency Management

| Requirement | Description | Verification |
| --- | --- | --- |
| SC-DEP-001 | All dependencies enumerated in SBOM (SPDX 2.3 or CycloneDX) | SBOM generation verification |
| SC-DEP-002 | Dependencies scanned for known vulnerabilities (CVE) | CVE scan report per build |
| SC-DEP-003 | Dependencies pinned to specific version hashes | Lockfile audit |
| SC-DEP-004 | Dependency updates require security review | Change review record |
| SC-DEP-005 | No unpinned/unvetted dependencies in release builds | Build config audit |

---

## 4. Firmware Signing Infrastructure

### 4.1 Signing Key Protection

| Requirement | Description |
| --- | --- |
| SC-SIG-001 | Firmware signing keys must be stored in an HSM (Hardware Security Module) |
| SC-SIG-002 | Signing operation must require multi-party authorization (quorum) |
| SC-SIG-003 | Signing must occur in an isolated (offline/air-gapped) environment |
| SC-SIG-004 | Signing keys must never leave the HSM (all operations within HSM boundary) |
| SC-SIG-005 | All signing operations must be logged with timestamp and operator identity |

### 4.2 Key Ceremony Procedure

| Step | Action | Participants |
| --- | --- | --- |
| 1 | HSM initialization | Minimum 2 security officers |
| 2 | Root key generation (if applicable) | Minimum 2 security officers |
| 3 | Signing key generation | Security officer + build engineer |
| 4 | Key backup (sharded, m-of-n scheme) | Security officers |
| 5 | Key activation (unlock for use) | Authorized operator |
| 6 | Key revocation (compromise response) | Security officer |

### 4.3 Signing Operation

| Step | Action | Verification |
| --- | --- | --- |
| 1 | Build artifact hash computed | Verify hash matches build output |
| 2 | Build provenance verified (reproducible build check) | Compare with independent build |
| 3 | Artifact submitted to HSM for signing | Operator authentication |
| 4 | HSM signs artifact | Quorum authorization (if configured) |
| 5 | Signature appended to image | Verify signature format per `Image_Format.md` |
| 6 | Signed artifact registered in firmware registry | Audit log entry created |

---

## 5. Distribution Security

### 5.1 Firmware Distribution Channel

| Requirement | Description |
| --- | --- |
| SC-DIST-001 | Firmware distribution channel must enforce TLS 1.3 with mutual authentication |
| SC-DIST-002 | Firmware download must include integrity verification (hash check) before installation |
| SC-DIST-003 | Distribution server must verify uploaded firmware signature before serving |
| SC-DIST-004 | Firmware metadata (version, hash, signature) must be served over authenticated channel |
| SC-DIST-005 | Firmware repository must support content-addressable storage (hash-verified retrieval) |

### 5.2 Firmware Mirror and CDN Security

| Requirement | Description |
| --- | --- |
| SC-MIRROR-001 | Mirror servers must verify content hash before serving |
| SC-MIRROR-002 | Edge/CDN cache poisoning must be detectable (hash mismatch = reject) |
| SC-MIRROR-003 | Firmware download must include origin verification on device |

---

## 6. Software Bill of Materials (SBOM)

### 6.1 SBOM Requirements

| Requirement | Description |
| --- | --- |
| SC-SBOM-001 | SBOM must be generated automatically as part of the build process |
| SC-SBOM-002 | SBOM must use standard format: SPDX 2.3 or CycloneDX 1.4+ |
| SC-SBOM-003 | SBOM must include all direct and transitive dependencies |
| SC-SBOM-004 | SBOM must include SBOP component itself (bootloader, crypto lib, OTA manager) |
| SC-SBOM-005 | SBOM must be signed and included in the firmware metadata |
| SC-SBOM-006 | SBOM must be verifiable on the device (hash check) |

### 6.2 SBOM Content

| Field | Description |
| --- | --- |
| Component name | SBOP firmware |
| Component version | From image header firmware_version |
| Supplier | Organization producing the firmware |
| Dependency list | All libraries, toolchains, and their versions |
| Hash algorithm | SHA-256 |
| Component hash | Hash of each component |
| License | License for each component |
| CVE status | Known vulnerabilities at build time |

---

## 7. Manufacturing Security Integration

→ See `Manufacturing_Security.md` for detailed manufacturing security controls.

Key supply chain integration points with manufacturing:

| Integration Point | Supply Chain Requirement |
| --- | --- |
| Provisioning station | Must verify firmware signature before flashing |
| Key injection | Must use HSM; keys must never be in plaintext |
| Device registration | Must record provisioning firmware version and hash |
| Fuse programming | Must verify successful lock before device ships |

---

## 8. Supply Chain Incident Response

| Incident | Response |
| --- | --- |
| Signing key compromise | Immediate key revocation, re-sign all firmware with new key, force-update all devices |
| Build environment compromise | Invalidate all builds from affected window, rebuild from clean environment |
| Dependency vulnerability (CVE) | Assess impact, patch dependency, rebuild, deploy via OTA |
| Distribution channel compromise | Block distribution, verify all served firmware integrity, rotate distribution keys |
| Counterfeit hardware detection | Verify device identity against provisioning database; reject unknown devices |

---

## 9. Requirements Trace

| Supply Chain Threat | SBOP Requirement | Control | Verification |
| --- | --- | --- | --- |
| SC-T-001 (Malicious commit) | SC-SRC-001 through SC-SRC-004 | Commit signing, review requirements | Audit |
| SC-T-002 (Build compromise) | SC-BLD-001 through SC-BLD-005 | Reproducible build, isolation | Independent build verification |
| SC-T-003 (Dependency attack) | SC-DEP-001 through SC-DEP-005 | SBOM, CVE scanning | SBOM audit, vulnerability scan |
| SC-T-004 (Signing key compromise) | SC-SIG-001 through SC-SIG-005 | HSM, quorum, air-gap | Key ceremony audit |
| SC-T-005 (Distribution compromise) | SC-DIST-001 through SC-DIST-005 | TLS, integrity verification | Distribution audit |
| SC-T-006 (Manufacturing threat) | REQ-SR-MFR-001, REQ-SR-MFR-002 | Manufacturing controls | Factory audit |
| SC-T-007 (Counterfeit hardware) | REQ-SR-ID-001, REQ-SR-ID-002 | Device identity | Identity verification |
| SC-T-008 (Unauthorized build) | SC-BLD-001, SC-SIG-001 | Build provenance, signing | Build audit |
