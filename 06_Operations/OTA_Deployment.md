# OTA Deployment Strategy

**Document ID:** OPS-OTA-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the safe, controlled rollout of firmware updates to SBOP devices. This includes deployment stages, rollout control mechanisms, safety guardrails, abort criteria, and integration with the dual-image A/B slot model for automatic rollback.

---

## 2. Deployment Pipeline

### 2.1 Stage Gate Model

```
Build → Internal Test → Canary → Staged Rollout → Full Deployment → Monitor
  │         │             │            │               │              │
  │    5 devices     0.1% fleet    1% → 10% →      100% fleet    30-day
  │    minimum       (≥ 100)       50% → 100%                     observation
```

### 2.2 Stage Details

| Stage | Fleet % | Min Devices | Duration | Pass Criteria | Fail Action |
| --- | --- | --- | --- | --- | --- |
| Internal test | N/A | 5 | 24 hours | All pass boot + OTA | Fix and restart |
| Canary | 0.1% | 100 | 48 hours | < 1% rollback, 0 tamper events | Halt, investigate |
| Staged 1 | 1% | 1,000 | 72 hours | < 1% rollback, < 0.1% boot fail | Halt, investigate |
| Staged 2 | 10% | — | 72 hours | < 0.5% rollback, < 0.05% boot fail | Halt, investigate |
| Staged 3 | 50% | — | 72 hours | < 0.5% rollback | Halt, investigate |
| Full | 100% | — | Indefinite | Monitoring continues | Rollback if thresholds exceeded |

---

## 3. Rollout Control

### 3.1 Backend Deployment Controller

```
DeploymentConfig {
    firmware_version: u32,
    target_fleet: FleetSelector,     // device cohort filter
    max_devices: u32,                // cap for staged rollout
    start_time: u64,                 // earliest download time
    expiry_time: u64,                // deadline (devices after this skip)
    rollback_version: u32,           // fallback version on failure
    require_ack: bool,               // device must confirm before install
    allow_downgrade: bool,           // override anti-rollback (emergency only)
    created_by: OperatorID,
    approved_by: [OperatorID; 2],    // dual approval required
}
```

### 3.2 Device-Side Eligibility Check

```
Device checks before OTA download:
  1. Device UID in target fleet?         → No → skip
  2. Current time < expiry_time?         → No → skip
  3. New version > current version?      → No → skip (unless allow_downgrade)
  4. Slot available (INACTIVE or EMPTY)? → No → wait or skip
  5. Battery level > minimum?            → No → defer
  6. Network connectivity adequate?      → No → defer
```

---

## 4. Safety Guardrails

### 4.1 Automatic Rollback

```
Boot detects PENDING slot never reached ACTIVE:
  1. Mark PENDING slot as INVALID
  2. Switch to FALLBACK slot (previous ACTIVE)
  3. Increment slot_fallback_count
  4. Boot fallback image
  5. If fallback succeeds: report rollback to backend via telemetry
  6. If fallback fails: enter FAILSAFE → await backend recovery OTA
```

### 4.2 Abort Conditions

| Condition | Threshold | Action |
| --- | --- | --- |
| Rollback rate | > 2% of updated devices | Immediate halt of deployment |
| Boot failure rate | > 1% of updated devices | Immediate halt |
| Tamper events | Any tamper event correlated with update | Immediate halt + SEV-1 |
| Critical bug report | Validated P0 bug in new version | Halt + assess |
| Backend anomaly | Unexpected registration pattern | Halt + investigate |

### 4.3 Emergency Rollback

```
Trigger: SEV-0 or SEV-1 incident related to current firmware

Procedure:
  1. Security architect authorizes emergency rollback
  2. Backend sets deployment status = EMERGENCY_ROLLBACK
  3. All devices instructed to revert to last known-good version
  4. Current version added to revocation list (block re-installation)
  5. Devices that already installed current version:
     a. If previous slot still valid → immediate boot to fallback
     b. If both slots compromised → force OTA of known-good version
  6. Monitor rollback completion across fleet
  7. Incident investigation proceeds per Incident_Response.md
```

---

## 5. Fleet Segmentation

### 5.1 Cohort Definitions

| Cohort | Description | Selection Criteria |
| --- | --- | --- |
| Development | Engineering devices | Pre-registered device list |
| Internal Dogfood | Company-operated devices | Separate fleet selector |
| Early Access | Customer volunteers | Opt-in flag |
| Production - Region A | Geographic segment | Device registration region |
| Production - Region B | Geographic segment | Device registration region |
| High-Risk | Safety-critical deployments | Deployment flag |
| Low-Risk | Non-critical deployments | Deployment flag |

### 5.2 Cohort Deployment Order

```
Internal Dogfood → Early Access → Low-Risk Production → High-Risk Production
                                                          (last, after ≥ 2 weeks of low-risk success)
```

---

## 6. Monitoring During Deployment

### 6.1 Real-Time Metrics

| Metric | Dashboard Refresh | Alert Threshold |
| --- | --- | --- |
| Devices updated (count + %) | 1 min | — |
| OTA success rate | 1 min | < 98% |
| Boot success post-update | 5 min | < 99% |
| Rollback rate | 5 min | > 1% |
| Tamper events | Real-time | Any |
| Error distribution (by code) | 15 min | Any new error code spike |

### 6.2 Post-Deployment Observation

After 100% deployment, elevated monitoring for 30 days:

| Week | Monitoring Intensity |
| --- | --- |
| Week 1 | Full real-time monitoring; daily review |
| Week 2 | Standard monitoring; daily review |
| Weeks 3-4 | Standard monitoring; weekly review |
| After 30 days | Baseline monitoring; version marked "stable" |

---

## 7. Deployment Approval Workflow

### 7.1 Required Approvals

| Stage | Approvals Required |
| --- | --- |
| Internal test → Canary | QA lead + release manager |
| Canary → Staged 1 | QA lead + release manager + security architect (if security changes) |
| Staged 1 → Staged 2 | Above + product manager |
| Staged 2 → Staged 3 | Above + compliance officer (if regulated deployment) |
| Staged 3 → Full | Above + VP Engineering |
| Emergency rollback | Security architect (single approval for speed) |

### 7.2 Approval Record

```
DeploymentApproval {
    deployment_id: UUID,
    stage: DeploymentStage,
    approver: OperatorID,
    timestamp: u64,
    signature: Signature,           // approver's digital signature
    conditions: [String],           // any conditions on approval
}
```

---

## 8. Rollback and Recovery Scenarios

### 8.1 Scenario: Corrupted OTA Image

```
1. Device downloads image → hash verification fails (OTA phase)
2. Device discards image, sets slot to INVALID
3. Device reports ERR-OTA-VERIFY-HASH to backend
4. Backend increments failure counter for version
5. If failure rate > threshold → halt deployment
6. Device retries download (exponential backoff: 1h, 2h, 4h, 8h)
7. If all retries fail → device stays on current version, reports to backend
```

### 8.2 Scenario: Boot Failure After Successful OTA

```
1. Device downloads, verifies, installs new image in slot B
2. Slot B marked PENDING; device reboots
3. Boot verifies slot B → fails (signature/hash/version)
4. Boot marks slot B INVALID, switches to slot A (previous ACTIVE)
5. Boot increments fallback_count, boots slot A
6. Device reports rollback to backend with error code
7. Backend counts rollback; if > threshold → halt deployment
```

### 8.3 Scenario: Power Loss During Slot Write

```
1. Device writing new image to slot B
2. Power lost mid-write
3. On power restoration, boot detects slot B in inconsistent state
4. Slot metadata CRC fails → slot marked INVALID
5. Boot selects slot A (still ACTIVE), boots normally
6. OTA client detects slot B is INVALID, requests re-download
7. Re-download begins from byte 0 (no resume — clean start)
```

---

## 9. Version Lifecycle

```
DRAFT → TEST → CANARY → STAGED → ACTIVE → SUPERSEDED → DEPRECATED → REVOKED
  │       │       │        │         │           │             │          │
  │   Internal  Canary  Staged   Full      Newer          No new       Blocked
  │   testing   fleet   rollout  fleet     version        installs     everywhere
  │                                   installed           allowed
```

---

## 10. References

| Document | Reference |
| --- | --- |
| OTA Flow Pseudocode | `../03_Subsystem/Update/OTA_Flow_Pseudocode.md` |
| OTA Recovery | `../03_Subsystem/Update/OTA_Recovery.md` |
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
| Incident Response | `Incident_Response.md` |
| Monitoring and Telemetry | `Monitoring_and_Telemetry.md` |
| Error Code Catalog | `../02_System_Design/Error_Code_Catalog.md` |
