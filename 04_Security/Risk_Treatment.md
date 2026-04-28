# Risk Treatment Strategy

**Document ID:** SEC-RISK-TRT-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines how identified risks are treated: mitigated, accepted, transferred, or avoided. Each treatment decision includes rationale, residual risk assessment, and verification that the treatment is effective.

→ Input: `Risk_Quantification.md` (risk scores), `TARA_Methodology.md` (threat scenarios)
→ Output: `Residual_Risk.md` (accepted risks), `Mitigation_Strategy.md` (mitigations)

---

## 2. Treatment Options

### 2.1 Treatment Definitions

| Treatment | Definition | When to Use |
| --- | --- | --- |
| **Mitigate** | Reduce risk via security controls | All risks above acceptance threshold |
| **Accept** | Acknowledge risk without additional controls | Residual risk below threshold; cost of mitigation exceeds benefit |
| **Transfer** | Shift risk to third party (insurance, contractual) | Business risk; not applicable to safety/security |
| **Avoid** | Eliminate the activity creating the risk | Risk too high to mitigate or accept |

### 2.2 Treatment Decision Matrix

| Impact | Feasibility: Low | Feasibility: Medium | Feasibility: High | Feasibility: Very High |
| --- | --- | --- | --- | --- |
| Critical (4) | **Avoid** or **Accept** (with continuous monitoring) | **Mitigate** aggressively | **Mitigate** | **Mitigate** |
| High (3) | **Accept** (monitored) | **Mitigate** | **Mitigate** | **Mitigate** |
| Medium (2) | **Accept** | **Accept** (monitored) | **Mitigate** (if cost-effective) | **Mitigate** |
| Low (1) | **Accept** | **Accept** | **Accept** | **Mitigate** (only if trivial) |

---

## 3. Risk Treatment Register

### 3.1 A1: Unauthorized Firmware Execution

| Leaf Node | Risk Score | Treatment | Rationale | Controls Applied |
| --- | --- | --- | --- | --- |
| A1.1.1 Skip verification (fault injection) | Critical (12) | Mitigate | Direct path to untrusted execution | Redundant verification, glitch detection, tamper response |
| A1.1.2 Exploit parsing bug | High (9) | Mitigate | Parser is first line of defense | Input validation, fuzzing, no fallback parsing |
| A1.2.1 Break crypto | Low (3) | Accept (monitored) | Standard algorithms (ECDSA, Ed25519) with no known practical breaks | CVE monitoring, crypto agility |
| A1.2.2 Steal signing key | High (12) | Mitigate | Key compromise = full compromise | HSM, quorum, air-gapped signing |
| A1.3.1 Tamper during transfer | Medium (6) | Mitigate | MITM is feasible attack | End-to-end signing, device-side verification |
| A1.3.2 Backend compromise | High (9) | Mitigate | Backend controls fleet | Access control, audit logging, deployment policy |

### 3.2 A4: Device Cloning

| Leaf Node | Risk Score | Treatment | Rationale | Controls Applied |
| --- | --- | --- | --- | --- |
| A4.1 Extract keys from device | High (9) | Mitigate | Breaks per-device identity | Secure element / TEE, physical tamper |
| A4.2 Copy identity data | Medium (6) | Mitigate | Enables cloning at scale | PUF/HUK binding, backend dedup |

### 3.3 A6: Supply Chain Compromise

| Leaf Node | Risk Score | Treatment | Rationale | Controls Applied |
| --- | --- | --- | --- | --- |
| A6.1.1 Malicious commit (insider) | High (9) | Mitigate | Insider threat is real | Commit signing, mandatory review, two-person rule |
| A6.2.1 Build server compromise | High (9) | Mitigate | Build server is high-value target | Reproducible builds, ephemeral environment |
| A6.3.1 Steal signing key from HSM | Critical (12) | Mitigate | Game over if successful | HSM access control, quorum, audit logging |
| A6.3.2 Sign unauthorized via insider | High (9) | Mitigate | Collusion risk | Multi-party signing, ceremony, audit |
| A6.4.1 Replace firmware on CDN | Medium (6) | Mitigate | CDN is outside SBOP trust boundary | TLS + device-side signature + hash |

### 3.4 A8: Side-Channel Attack

| Leaf Node | Risk Score | Treatment | Rationale | Controls Applied |
| --- | --- | --- | --- | --- |
| A8.1.1 Timing oracle on signature | Medium (6) | Mitigate | Remote timing is noisy but real | Constant-time ECDSA/Ed25519 |
| A8.1.2 Timing oracle on hash compare | Medium (6) | Mitigate | Local timing oracle feasible | `constant_time_compare` |
| A8.2.1 SPA on ECDSA | Medium (4) | Mitigate at SL 3+ | Requires physical access + equipment | Fixed-sequence ops, balanced point ops |
| A8.2.2 DPA on key derivation | High (6) | Mitigate at SL 3+ | High-trace-count attack | Masking, blinding |

### 3.5 A11: Zone Boundary Violation

| Leaf Node | Risk Score | Treatment | Rationale | Controls Applied |
| --- | --- | --- | --- | --- |
| A11.1.1 App reads boot memory | High (9) | Mitigate | Zone 2 compromise must not escalate | MPU/MMU, Zone 1 locked read-only |
| A11.1.2 OTA writes to active slot | Medium (6) | Mitigate | Could bypass boot verification | Boot slot selection in Zone 1 only |

---

## 4. Residual Risk Assessment

After mitigation, each risk is re-assessed:

| Original Risk | Mitigation Effectiveness | Residual Impact | Residual Feasibility | Residual Score | Decision |
| --- | --- | --- | --- | --- | --- |
| A1.1.1 (Critical, 12) | High — redundant checks + glitch detection | Critical (4) | Very Low (1) | 4 | Accept residual (monitored) |
| A1.2.2 (Critical, 12) | Very High — HSM + quorum + air gap | Critical (4) | Very Low (1) | 4 | Accept residual |
| A6.3.1 (Critical, 12) | High — HSM + quorum + audit | Critical (4) | Very Low (1) | 4 | Accept residual (RRES-SC-001) |
| A8.2.2 (High, 6) at SL 1-2 | Low at SL 1-2 (no masking) | High (3) | Low (2) at SL 1-2 | 6 at SL 1-2 | Accept for SL ≤ 2 (RRES-SCH-001) |

→ See `Residual_Risk.md` for the complete residual risk register with 29 entries.

---

## 5. Treatment Verification

### 5.1 Verification Methods

| Verification | Method | When |
| --- | --- | --- |
| Design review | Security architect reviews mitigation design | Pre-implementation |
| Implementation review | Code review of mitigation | Post-implementation |
| Functional test | Verify mitigation works (normal + error paths) | Per test plan |
| Penetration test | Red team attempts to bypass mitigation | Pre-release |
| Audit | Process audit for procedural controls | Periodic |

### 5.2 Effectiveness Measurement

Each mitigation is assigned an effectiveness rating post-verification:

| Rating | Definition |
| --- | --- |
| Very High | No known bypass; defense-in-depth (multiple independent controls) |
| High | Single strong control; no practical bypass known |
| Medium | Control effective but has known limitations (documented in residual risk) |
| Low | Control provides partial mitigation; significant residual risk (requires acceptance) |

---

## 6. Risk Acceptance Process

### 6.1 Acceptance Criteria

A risk may be accepted only when:
1. Feasibility score is ≤ 2 (Low or Very Low), OR
2. Cost of mitigation exceeds the risk exposure (business decision), AND
3. Safety is not compromised (no acceptance if safety goal is violated), AND
4. Acceptance is documented with monitoring controls

### 6.2 Acceptance Sign-Off

| Risk Level | Required Sign-Off |
| --- | --- |
| Residual Critical (score ≥ 9) | Security Architect + Product Manager + Compliance Officer |
| Residual High (score 6-8) | Security Architect + Product Manager |
| Residual Medium (score 3-5) | Security Architect |
| Residual Low (score 1-2) | Documented in register (no formal sign-off) |

---

## 7. Continuous Risk Management

### 7.1 Triggers for Re-Assessment

- New attack technique published (e.g., new fault injection method)
- Vulnerability discovered in dependency (CVE)
- Architecture change
- Deployment in higher-safety context
- Annual security review

### 7.2 Risk Metrics

| Metric | Target |
| --- | --- |
| Risks with residual score ≥ 9 | 0 |
| Risks with residual score ≥ 6 (High) | ≤ 5 |
| Accepted risks without monitoring | 0 |
| Unmitigated critical risks | 0 |

---

## 8. References

| Document | Reference |
| --- | --- |
| Risk Quantification | `Risk_Quantification.md` |
| Residual Risk Register | `Residual_Risk.md` |
| Mitigation Strategy | `Mitigation_Strategy.md` |
| TARA Methodology | `TARA_Methodology.md` |
| Attack Tree | `Attack_Tree.md` |
