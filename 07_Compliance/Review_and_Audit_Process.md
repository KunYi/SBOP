# Review and Audit Process

**Document ID:** CMP-AUD-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the internal review and external audit procedures for SBOP. Covers design reviews, security assessments, compliance audits, and continuous assurance activities across all 6 supported standards.

---

## 2. Review Types

### 2.1 Internal Reviews

| Review Type | Frequency | Participants | Focus |
| --- | --- | --- | --- |
| Design review | Per design change | Security architect, subsystem owner | Architecture correctness, security properties |
| Code review | Per commit | Peer developer, security reviewer | Implementation correctness, no vulnerabilities |
| Security review | Per release | Security architect | Threat model update, new attack surface |
| Test review | Per test cycle | QA lead, security architect | Coverage gaps, test quality |
| Risk review | Quarterly | Security architect, product manager | New risks, residual risk status |
| Specification review | Per spec change | Systems engineer, security architect | Consistency, completeness |

### 2.2 External Audits

| Audit Type | Frequency | Standard | Auditor |
| --- | --- | --- | --- |
| ISO 21434 compliance | Per program phase | ISO 21434 | Accredited assessor |
| IEC 62443 certification | Per product | IEC 62443-4-1/4-2 | ISASecure or equivalent |
| DO-178C SOI audit | Per DAL level | DO-178C/DO-330 | DER/FAA designee |
| IEC 62304 audit | Per device class | IEC 62304 | Notified body (EU MDR) |
| ECSS PA audit | Per mission | ECSS Q-ST-80 | ESA/CNES PA |
| Penetration test | Annually | N/A | Third-party security lab |

---

## 3. Internal Review Process

### 3.1 Design Review

```
Entry Criteria:
  - Design document complete (all sections, cross-references)
  - Interfaces specified
  - Error handling defined

Process:
  1. Author submits design for review (with review checklist)
  2. Security architect reviews: threat model coverage, security properties
  3. Subsystem owner reviews: interface contracts, state machine correctness
  4. Systems engineer reviews: consistency with system specification
  5. Comments resolved; author updates document
  6. Re-review if substantial changes
  7. Sign-off: security architect + subsystem owner + systems engineer

Exit Criteria:
  - All review comments resolved
  - No open blocking issues
  - Design traceable to requirements
```

### 3.2 Security Review Checklist

| # | Check | Pass/Fail |
| --- | --- | --- |
| 1 | All attack tree leaf nodes have mitigations | |
| 2 | All security requirements have verification tests | |
| 3 | Interfaces do not expose raw key material | |
| 4 | All crypto operations are constant-time or justified | |
| 5 | Error paths lead to safe state (FAILSAFE or rejection) | |
| 6 | No debug bypass path exists | |
| 7 | Zone 1/Zone 2 memory isolation is enforced | |
| 8 | Anti-rollback is not bypassable | |
| 9 | Supply chain controls are documented | |
| 10 | All new dependencies reviewed for CVEs | |
| 11 | SBOM updated | |
| 12 | Residual risks reviewed and accepted | |

### 3.3 Code Review Requirements

| Requirement | Mandatory for |
| --- | --- |
| Two-person review | All security-critical code (boot, crypto, OTA) |
| Security reviewer | All code touching keys, verification, state machine |
| Static analysis pass | All code (zero warnings on critical path) |
| Fuzz test pass | All parsers (see Fuzzing_Strategy.md) |

---

## 4. Audit Preparation

### 4.1 Evidence Package

The evidence package includes all documentation and test results needed to demonstrate compliance. It is organized by standard and clause.

→ See `Evidence_List.md` for the complete evidence catalog.

### 4.2 Pre-Audit Checklist

```
□ All requirements have trace chains (REQ → Design → Impl → Test → Evidence)
□ No broken cross-references in specification documents
□ All test results current (run within audit window)
□ All accepted risks reviewed and re-confirmed
□ Residual risk register up to date
□ Incident response records available (if any)
□ Key management audit logs complete
□ SBOM current for all firmware versions
□ Coverage reports generated and reviewed
□ Reviewer sign-offs documented
```

### 4.3 Audit Readiness Dashboard

| Metric | Green | Yellow | Red |
| --- | --- | --- | --- |
| Requirement coverage | 100% | 95-99% | < 95% |
| Test pass rate | 100% | 98-99% | < 98% |
| Open security issues | 0 critical/high | 1-3 medium | Any critical |
| Document review currency | All within 12 months | Within 18 months | > 18 months |
| CVE status in dependencies | 0 unpatched critical/high | 1-3 medium | Any critical |

---

## 5. External Audit Procedure

### 5.1 Audit Phases

```
Phase 1: Planning
  - Define audit scope (which standards, which clauses)
  - Agree on evidence requirements
  - Schedule audit dates
  - Identify participants

Phase 2: Evidence Submission
  - Provide evidence package to auditor
  - Answer preliminary questions
  - Auditor reviews evidence off-site (2-4 weeks)

Phase 3: On-Site Audit
  - Auditor visits facility (or remote equivalent)
  - Live demonstrations: boot verification, OTA update, tamper response
  - Interviews: security architect, developers, operations
  - Tool chain review: build pipeline, HSM setup, signing ceremony
  - Evidence trace walkthrough: requirement → design → code → test → result

Phase 4: Findings and Remediation
  - Auditor issues findings report
  - Findings classified: Major NC, Minor NC, Observation, Opportunity
  - Remediation plan created with timeline
  - Corrective actions implemented
  - Evidence of correction submitted

Phase 5: Certification
  - Auditor confirms all findings resolved
  - Certificate or compliance report issued
  - Surveillance schedule established
```

### 5.2 Finding Classification

| Class | Definition | Response Timeline | Example |
| --- | --- | --- | --- |
| Major Non-Conformity (NC) | Systematic failure to meet requirement; no effective control | 30 days | No anti-rollback mechanism implemented |
| Minor NC | Isolated lapse; control exists but evidence incomplete | 90 days | Missing test result for one requirement |
| Observation | Not a failure, but improvement recommended | Next release | Comments in code could be clearer |
| Opportunity for Improvement (OFI) | Suggestion beyond minimum requirements | Consideration | Adopt newer crypto algorithm |

---

## 6. Continuous Compliance

### 6.1 Change Impact Assessment

When any change occurs, assess impact on:

| Change | Impact Assessment |
| --- | --- |
| New firmware version | Update SBOM, re-run tests, update evidence |
| Dependency update | CVE scan, re-verify affected interfaces |
| Architecture change | Re-assess threat model, re-assess all risks |
| New deployment context | Assess against relevant standards, update compliance mapping |
| Security incident | Root cause analysis, update controls, update residual risk |
| Standard update (new version) | Gap analysis against new version, update mappings |

### 6.2 Surveillance Audits

Post-certification, periodic surveillance ensures continued compliance:

| Standard | Surveillance Frequency | Scope |
| --- | --- | --- |
| ISO 21434 | Annual | Changes since last audit, incident review |
| IEC 62443 | Annual | Changes, vulnerability management |
| DO-178C | Per major change | Affected DAL components |
| IEC 62304 | Annual (Class C) | Changes, post-market surveillance |
| ECSS | Per mission milestone | Phase-dependent |

---

## 7. Roles and Responsibilities

| Role | Internal Review | External Audit |
| --- | --- | --- |
| Security Architect | Conducts security reviews; signs off | Technical SME for auditor |
| System Engineer | Conducts design reviews | Traceability demonstration |
| QA Lead | Conducts test reviews | Test evidence presentation |
| Product Manager | Approves risk acceptance | Business context |
| Compliance Officer | Ensures evidence readiness | Audit liaison |
| Developer | Code review participant | Code walkthrough |

---

## 8. Document Control

### 8.1 Review Records

All reviews produce a signed record:

```
Review Record
=============
Document: [name + version]
Review Type: [Design / Security / Code / Test]
Review Date: YYYY-MM-DD
Reviewers: [names + roles]
Findings: N (0 critical, 0 major, N minor)
Disposition: Approved / Approved with Comments / Rejected
Signatures: [reviewers]
```

### 8.2 Retention

| Record | Retention |
| --- | --- |
| Design review records | Life of product + 10 years |
| Security review records | Life of product + 10 years |
| Audit reports | Life of product + 10 years |
| Finding remediation records | Life of product + 10 years |
| Certificate / compliance letters | Permanent |

---

## 9. References

| Document | Reference |
| --- | --- |
| Evidence List | `Evidence_List.md` |
| Work Products | `Work_Products.md` |
| Audit Trace | `Audit_Trace.md` |
| Compliance Overview | `Compliance_Overview.md` |
| ISO 21434 Detailed Mapping | `ISO_21434_Detailed_Mapping.md` |
| IEC 62443 Detailed Mapping | `IEC_62443_Detailed_Mapping.md` |
| ECSS Compliance Mapping | `ECSS_Compliance_Mapping.md` |
| Medical Compliance Mapping | `Medical_Compliance_Mapping.md` |
| Aviation Compliance Mapping | `Aviation_Compliance_Mapping.md` |
