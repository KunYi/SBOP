# Incident Response Plan

**Document ID:** OPS-IR-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the detection, response, containment, and recovery procedures for security and operational incidents affecting SBOP-protected devices or SBOP infrastructure.

---

## 2. Incident Classification

### 2.1 Severity Levels

| Severity | Definition | Examples | Response SLA |
| --- | --- | --- | --- |
| SEV-0 (Critical) | Active compromise of signing infrastructure or root key | HSM breach, KR compromise, unauthorized firmware signed | Immediate (15 min) |
| SEV-1 (High) | Vulnerability enabling device compromise at scale | Verification bypass discovered, debug auth broken, side-channel key recovery feasible | 4 hours |
| SEV-2 (Medium) | Vulnerability affecting single device or requiring physical access | Physical tamper of individual device, single-device debug unlock | 24 hours |
| SEV-3 (Low) | Non-critical anomaly or policy violation | Missed audit log entry, failed provisioning retry above threshold | 5 business days |

### 2.2 Incident Types

| Type | Description | Related Attack Tree |
| --- | --- | --- |
| IR-TYPE-01 | Compromised firmware in distribution | A6 (Supply Chain Compromise) |
| IR-TYPE-02 | Signing key compromise or suspected compromise | A6.3 (Steal signing key) |
| IR-TYPE-03 | Mass OTA failure or rollback | A5 (OTA Exploitation) |
| IR-TYPE-04 | Device cloning detected at scale | A4 (Device Cloning), A10 (Manufacturing Bypass) |
| IR-TYPE-05 | Side-channel attack demonstrated | A8 (Side-Channel Attack) |
| IR-TYPE-06 | Physical tampering campaign | A7 (Physical Tampering) |
| IR-TYPE-07 | Debug authentication bypass | A9 (Debug Exploitation) |
| IR-TYPE-08 | Backend infrastructure compromise | A11.3 (Infrastructure compromise) |
| IR-TYPE-09 | Zero-day vulnerability in crypto dependency | RRES-SC-002 |
| IR-TYPE-10 | Insider threat detected | RRES-SC-001 |

---

## 3. Response Process

### 3.1 Phase 1: Detection

**Detection Sources:**

| Source | What It Detects | Alert Threshold |
| --- | --- | --- |
| Backend anomaly detection | Abnormal OTA patterns, duplicate UIDs, unusual rollback rates | Configurable per fleet |
| Device telemetry | Boot failures, tamper events, debug auth attempts | Per-device, aggregated |
| CVE monitoring | Vulnerabilities in dependencies (mbedTLS, wolfSSL, libsodium) | Any critical/high CVE |
| HSM audit log | Unauthorized signing attempts, quorum failures | Any denied operation |
| Signing ceremony log | Irregularities in ceremony procedure | Any deviation |
| Build pipeline | Non-reproducible build, unauthorized commit | Any non-reproducibility |
| External report | Researcher disclosure, customer report, threat intelligence | Any credible report |

### 3.2 Phase 2: Triage and Assessment

```
1. Confirm incident: Is this a real security incident or a false positive?
2. Classify severity: Apply severity definitions from §2.1
3. Determine scope:
   - How many devices affected?
   - Which firmware versions?
   - Is signing infrastructure affected?
   - Is the root key (KR) affected?
4. Assign incident commander (SEV-0/1: security architect; SEV-2: ops lead; SEV-3: on-call)
5. Open incident record with initial assessment
```

### 3.3 Phase 3: Containment

| Incident Type | Containment Action |
| --- | --- |
| Compromised firmware detected | Immediately revoke affected KI; halt all deployments using that KI |
| Signing key compromise (KI) | Revoke KI; halt all signing; transition to new KI |
| Root key compromise (KR) | Emergency KR rotation; all devices must be re-provisioned or updated with new KD derivation |
| Mass OTA failure | Halt deployment; force rollback to previous version on affected devices |
| Device cloning | Block duplicate UIDs at backend; investigate manufacturing logs |
| Infrastructure compromise | Isolate affected systems; rotate all backend credentials |
| Zero-day in dependency | Assess exploitability in SBOP context; plan patch timeline |

### 3.4 Phase 4: Investigation

```
Root cause analysis (RCA):
  1. Timeline reconstruction: What happened, when, in what order?
  2. Attack vector identification: How did the attacker gain access?
  3. Affected assets enumeration: What keys, firmware, devices were impacted?
  4. Evidence preservation: Secure logs, disk images, HSM audit trails
  5. Impact assessment: What is the worst-case impact on device safety/security?

For SEV-0/1 incidents:
  - RCA must be completed within 72 hours
  - External forensic firm engaged if criminal activity suspected
  - Legal counsel notified for potential disclosure obligations
```

### 3.5 Phase 5: Remediation

| Incident | Remediation |
| --- | --- |
| KI compromise | Sign all active firmware versions with new KI; force update to re-signed firmware |
| KR compromise | Full re-provisioning of all devices (field return or remote if possible) |
| Vulnerability in verification | Patch bootloader; deploy via OTA; devices that cannot be patched are deprecated |
| Device cloning at factory | Identify rogue factory; rotate provisioning credentials; re-provision legitimate devices |
| Zero-day in crypto lib | Vendor patch applied; re-verify all crypto operations; re-sign firmware with patched lib |

### 3.6 Phase 6: Recovery

```
1. Verify remediation: Confirm all affected devices have received fix
2. Monitor: Increased telemetry frequency for 2 weeks post-remediation
3. Restore normal operations: Re-enable deployments, unblock devices
4. Lessons learned: Post-incident review within 2 weeks
   - What failed? (process, technology, people)
   - What worked? (detection, response, containment)
   - What changes prevent recurrence?
```

---

## 4. Communication Plan

| Audience | SEV-0 | SEV-1 | SEV-2 | SEV-3 |
| --- | --- | --- | --- | --- |
| Security architect | Immediate | Immediate | 1 hour | 24 hours |
| Product manager | 1 hour | 4 hours | 24 hours | Next review |
| Compliance officer | 4 hours | 24 hours | 48 hours | Next review |
| Customers/integrators | Per contractual SLA | Per contractual SLA | Next update cycle | Not required |
| Regulators | As required by law | As required by law | As required by law | Not required |
| Public | Per disclosure policy | Per disclosure policy | Not required | Not required |

---

## 5. Incident-Specific Playbooks

### 5.1 Suspected KI Compromise

```
1. IMMEDIATE: Halt all signing operations
2. Verify: Review HSM audit logs for unauthorized signing
3. If confirmed:
   a. Revoke KI in backend → all devices → block images signed by that KI
   b. Generate new KI via emergency ceremony
   c. Re-sign all active firmware versions with new KI
   d. Force OTA of re-signed firmware to all devices
   e. If unable to determine when compromise began:
      - Assume all versions signed by compromised KI are untrusted
      - Re-sign from last known-good version
4. If not confirmed (false alarm):
   a. Document investigation findings
   b. Resume operations
   c. Review detection rule to reduce false positives
```

### 5.2 Mass Rollback Event

```
1. Detect: Backend telemetry shows rollback rate > threshold (e.g., > 2%)
2. Halt: Immediately stop deployment of current version
3. Investigate:
   a. Is the new image corrupted in distribution?
   b. Is there a device-specific compatibility issue?
   c. Is this a targeted attack causing verification failures?
4. If image corruption:
   a. Verify image hash at backend → re-sign if needed
   b. Re-deploy corrected image
5. If compatibility:
   a. Halt deployment to affected device cohorts
   b. Fix issue, test, re-deploy
6. If attack:
   a. Escalate to SEV-1
   b. Follow §5.1 if signing integrity in question
```

### 5.3 Backend Infrastructure Compromise

```
1. Isolate: Disconnect affected systems from network
2. Assess: Which systems? What data accessed? What credentials exposed?
3. Rotate: All backend credentials, API keys, TLS certificates
4. Verify: Device registry integrity (any unauthorized device registrations?)
5. Rebuild: Affected systems from known-good images
6. Harden: Apply additional controls to prevent recurrence
7. Notify: All customers if device data was exposed (per GDPR/regulatory requirements)
```

---

## 6. Incident Record Template

| Field | Value |
| --- | --- |
| Incident ID | IR-YYYY-NNN |
| Date/Time Detected | YYYY-MM-DD HH:MM UTC |
| Detected By | System / Person |
| Severity | SEV-0/1/2/3 |
| Type | IR-TYPE-NN |
| Status | Open / Investigating / Contained / Resolved |
| Incident Commander | Name |
| Affected Systems | List |
| Affected Device Count | Number |
| Root Cause | Summary |
| Timeline | Key events with timestamps |
| Containment Actions | List |
| Remediation Actions | List |
| Lessons Learned | List |
| Closure Date | YYYY-MM-DD |

---

## 7. Team and Responsibilities

| Role | Responsibility | SEV-0/1 Escalation Path |
| --- | --- | --- |
| Incident Commander | Leads response; makes decisions | On-call rotation |
| Security Architect | Technical SME; assesses impact | Always notified for SEV-0/1 |
| Operations Lead | Executes containment and remediation | On-call rotation |
| Communications Lead | External/internal communications | PR/legal for SEV-0 |
| Legal Counsel | Regulatory disclosure; liability | SEV-0 only |

---

## 8. References

| Document | Reference |
| --- | --- |
| Attack Tree | `../04_Security/Attack_Tree.md` |
| Residual Risk Register | `../04_Security/Residual_Risk.md` |
| Key Management | `Key_Management.md` |
| Monitoring and Telemetry | `Monitoring_and_Telemetry.md` |
| Supply Chain Security | `../04_Security/Supply_Chain_Security.md` |
