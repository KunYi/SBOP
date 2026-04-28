# Security Identifier Rules

**Document ID:** SEC-IDR-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines identifier consistency rules across all security documents. Every identifier in the security domain must be unique, traceable, and consistently used across all documents.

---

## 2. Attack Tree ID Format

```
A<branch>.<sub>.<leaf>
```

| Level | Format | Example | Scope |
|-------|--------|---------|-------|
| Root | G-ROOT | G-ROOT | Compromise Device Trust |
| Branch | A<1-11> | A1 | Execute Unauthorized Firmware |
| Sub-branch | A<branch>.<sub> | A1.1 | Bypass Signature Verification |
| Leaf | A<branch>.<sub>.<leaf> | A1.1.1 | Skip verification (fault injection) |

**Consistency rule:** Same leaf node must use identical ID across `Attack_Tree.md`, `Risk_Quantification.md`, `Enhanced_Security_Trace.md`, `Mitigation_Strategy.md`, and `Security_Case.md`.

---

## 3. Requirement ID Format

```
REQ-<TYPE>-<DOMAIN>-<ID>
```

| TYPE | Domain | Example |
|------|--------|---------|
| FR | Functional Requirement | REQ-FR-BOOT-001 |
| SR | Security Requirement | REQ-SR-PHY-001 |
| NFR | Non-Functional Requirement | REQ-NFR-SYS-001 |

**Consistency rule:** Requirement IDs must be identical across `Requirements_Overview.md`, `Security_Trace.md`, `Enhanced_Security_Trace.md`, `Traceability_Matrix.md`, and `Audit_Trace.md`.

---

## 4. Test ID Format

```
<CATEGORY>-<DOMAIN>-<ID>
```

| Category | Domain | Example |
|----------|--------|---------|
| TEST | Functional test | TEST-BOOT-001 |
| FI | Fault injection | FI-001 |
| TIM | Timing measurement | TIM-001 |
| DBG-TEST | Debug test | DBG-TEST-001 |
| SC-AUDIT | Supply chain audit | SC-AUDIT-001 |
| NEG | Negative test | NEG-001 |

---

## 5. Error Code Format

```
ERR-<COMPONENT>-<CATEGORY>-<ID>
```

See `Error_Code_Catalog.md` for the complete registry.

---

## 6. Domain-Specific ID Prefixes

| Domain | ID Prefix | Example | Scope |
|--------|-----------|---------|-------|
| Architecture | ARC | ARC-CTX-001 | Top-level architecture documents |
| Requirements | REQ | REQ-FR-BOOT-001 | All requirements |
| System Design | SYS | SYS-DS-001 | System design documents |
| Subsystem: Boot | SUB-BOOT | SUB-BOOT-FLOW-001 | Boot subsystem docs |
| Subsystem: OTA | SUB-OTA | SUB-OTA-SM-001 | OTA subsystem docs |
| Subsystem: Identity | SUB-ID | SUB-ID-OV-001 | Identity subsystem docs |
| Subsystem: Crypto | SUB-CRYPTO | SUB-CRYPTO-ALG-001 | Crypto subsystem docs |
| Security | SEC | SEC-AT-001 | Security documents |
| Verification | VER | VER-STRAT-001 | Verification documents |
| Compliance | CMP | CMP-OV-001 | Compliance documents |
| Operations | OPS | OPS-KEY-001 | Operations documents |
| Product | PRD | PRD-REF-001 | Product documents |

---

## 7. Enforcement

Any mismatch in identifier usage across documents is considered a traceability failure and must be resolved before:
- Release Candidate (RC) gate
- Compliance audit submission
- Security certification review

Cross-reference validation is part of the `Review_and_Audit_Process.md`.

---

## 8. References

| Document | Reference |
|----------|-----------|
| Error Code Catalog | `../02_System_Design/Error_Code_Catalog.md` |
| Attack Tree | `Attack_Tree.md` |
| Security Trace | `Security_Trace.md` |
| Review Process | `../07_Compliance/Review_and_Audit_Process.md` |
