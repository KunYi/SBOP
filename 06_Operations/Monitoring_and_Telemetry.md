# Monitoring and Telemetry

**Document ID:** OPS-MON-001
**Version:** 2.0
**Status:** Draft
**Last Review:** 2026-04-28

---

## 1. Purpose

Defines the runtime observability framework for SBOP devices and backend infrastructure. Telemetry enables anomaly detection, fleet health monitoring, incident response, and continuous assurance of security controls.

---

## 2. Device Telemetry

### 2.1 Boot Event Telemetry

| Metric | Type | Description | Critical Threshold |
| --- | --- | --- | --- |
| `boot_count` | Counter | Total boot attempts (increments each RESET) | N/A |
| `boot_success_count` | Counter | Boots reaching EXECUTE | — |
| `boot_fail_count` | Counter | Boots entering FAILSAFE | > 3 consecutive → alert |
| `boot_fail_reason` | Enum | Last boot failure reason (error code) | — |
| `boot_duration_ms` | Gauge | Time from RESET to EXECUTE | > budget × 1.5 → warn |
| `slot_active` | Enum | Currently active slot (A or B) | — |
| `slot_fallback_count` | Counter | Times system fell back to previous slot | > 2 per version → alert |
| `firmware_version` | Uint32 | Currently executing firmware version | — |

### 2.2 OTA Event Telemetry

| Metric | Type | Description | Critical Threshold |
| --- | --- | --- | --- |
| `ota_attempt_count` | Counter | Total OTA attempts | — |
| `ota_success_count` | Counter | Successful OTA completions | — |
| `ota_fail_count` | Counter | Failed OTA attempts | > 3 consecutive → alert |
| `ota_fail_reason` | Enum | Last OTA failure reason | — |
| `ota_download_bytes` | Counter | Total bytes downloaded | — |
| `ota_rollback_count` | Counter | Rollbacks triggered | > 2% of fleet → alert |
| `ota_version_target` | Uint32 | Version being installed | — |

### 2.3 Security Event Telemetry

| Metric | Type | Description | Critical Threshold |
| --- | --- | --- | --- |
| `tamper_event_count` | Counter | Tamper sensor triggers | Any → SEV-1 alert |
| `tamper_event_type` | Enum | Type of tamper detected | — |
| `glitch_counter` | Counter | Fault injection attempts detected | > 0 → alert |
| `debug_auth_attempt_count` | Counter | Debug authentication attempts | > 3 in 24h → alert |
| `debug_auth_fail_count` | Counter | Failed debug auth attempts | ≥ 10 (lockout) → SEV-2 |
| `debug_unlock_duration_s` | Gauge | Duration of debug unlock (if authorized) | > authorized window → alert |
| `signature_verify_fail` | Counter | Signature verification failures | > 0 → alert |
| `hash_verify_fail` | Counter | Hash verification failures | > 0 → alert |
| `version_check_fail` | Counter | Anti-rollback rejections | > 0 → alert |

### 2.4 Health Telemetry

| Metric | Type | Description | Critical Threshold |
| --- | --- | --- | --- |
| `uptime_s` | Counter | Device uptime since last boot | — |
| `flash_error_count` | Counter | Flash read/write errors | > threshold → warn |
| `slot_metadata_crc_fail` | Counter | Slot metadata CRC failures | > 0 → alert |
| `watchdog_reset_count` | Counter | Watchdog-triggered resets | > 2 in 24h → warn |
| `battery_voltage_mv` | Gauge | Battery level (if applicable) | < minimum → warn |

---

## 3. Telemetry Data Structure

```
struct TelemetryRecord {
    header: TelemetryHeader,
    boot: BootTelemetry,
    ota: OTATelemetry,
    security: SecurityTelemetry,
    health: HealthTelemetry,
    crc32: u32,
}

struct TelemetryHeader {
    magic: u32 = 0x544C4D54 ("TLMT"),
    version: u16,
    device_uid_hash: [u8; 16],   // HMAC(KD, UID) — anonymized
    timestamp: u64,               // device time or monotonic counter
    sequence_number: u32,         // monotonically increasing
    record_length: u16,
}
```

---

## 4. Data Collection and Privacy

### 4.1 Collection Principles

| Principle | Implementation |
| --- | --- |
| Minimization | Only collect metrics needed for security and operations |
| Anonymization | Device identity hashed (HMAC with KD); backend correlates without exposing UID |
| No secrets in telemetry | Raw keys, hashes of current firmware, and debug tokens never included |
| Opt-out capability | Per deployment context (e.g., air-gapped systems disable telemetry) |

### 4.2 Data Sensitivity Classification

| Data Class | Examples | Handling |
| --- | --- | --- |
| Public | Version, uptime, slot health | Unrestricted |
| Internal | Boot success rate, OTA progress | Backend access only |
| Confidential | Anonymized device identity, tamper events | Restricted backend access, encrypted at rest |
| Secret | Raw keys, debug credentials | Never transmitted |

---

## 5. Backend Monitoring

### 5.1 Fleet-Level Metrics

| Metric | Aggregation | Alert Threshold |
| --- | --- | --- |
| Fleet boot success rate | Per version × per device cohort | < 99% → alert |
| Fleet OTA success rate | Per deployment | < 98% → alert, halt deployment at < 95% |
| Rollback rate | Per version | > 2% → halt deployment |
| Tamper event rate | Fleet-wide | Any cluster of tamper events → SEV-1 |
| Debug unlock rate | Per device cohort | Unexpected unlocks → investigate |
| Device duplication rate | Fleet-wide | Any duplicate UID hash → SEV-1 |
| Unregistered device connection | Fleet-wide | Any → alert |

### 5.2 Infrastructure Monitoring

| Component | Metric | Alert Threshold |
| --- | --- | --- |
| HSM | Signing operations/hour | Deviation from baseline |
| HSM | Failed auth attempts | > 0 → SEV-1 |
| Build pipeline | Non-reproducible builds | > 0 → SEV-1 |
| Firmware registry | Unauthorized upload attempt | > 0 → SEV-2 |
| Device registry | Registration rate anomaly | > 3σ from baseline |
| Backend API | TLS handshake failures | Spike → investigate |

---

## 6. Anomaly Detection

### 6.1 Detection Rules

| Rule ID | Description | Logic | Severity |
| --- | --- | --- | --- |
| ANOM-01 | Sudden increase in boot failures | `boot_fail_rate > baseline + 3σ` | SEV-2 |
| ANOM-02 | Duplicate anonymized device identity | `count(DISTINCT device_uid_hash) < count(records)` | SEV-1 |
| ANOM-03 | Version downgrade without rollback | `new_version < previous_version AND rollback_count = 0` | SEV-1 |
| ANOM-04 | Geographic anomaly | `device_location changes impossibly fast` | SEV-2 |
| ANOM-05 | OTA download from unexpected source | `ota_source_ip NOT IN allowed_cidr` | SEV-2 |
| ANOM-06 | Debug unlock outside maintenance window | `debug_unlock AND time NOT IN window` | SEV-2 |
| ANOM-07 | Rapid sequential boot failures | `boot_fail_count > 5 AND time_between < 60s` | SEV-2 |

---

## 7. Alerting and Escalation

### 7.1 Alert Routing

| Severity | Channel | Recipients | Ack SLA | Resolve SLA |
| --- | --- | --- | --- | --- |
| SEV-0 | Pager + phone + Slack | Security architect, ops lead | 15 min | 4 hours |
| SEV-1 | Pager + Slack | On-call engineer, security architect | 30 min | 24 hours |
| SEV-2 | Slack + email | On-call engineer | 4 hours | 5 days |
| SEV-3 | Email + dashboard | Operations team | Next business day | Next release |

### 7.2 Alert Suppression

Alerts are suppressed when:
- Device is in known maintenance window
- Device is in provisioning state (not yet deployed)
- Alert was already acknowledged and is being investigated (deduplication)
- Device is in a test fleet (non-production)

---

## 8. Telemetry Transport Security

| Layer | Mechanism |
| --- | --- |
| Transport | TLS 1.3 with mutual authentication (device KD_Auth → client cert) |
| Message integrity | HMAC-SHA256 over telemetry payload (keyed with KD_Auth) |
| Replay protection | Sequence number in TelemetryHeader; backend rejects duplicates |
| Backend storage | Encrypted at rest; retention per data classification policy |

---

## 9. Dashboards and Reporting

### 9.1 Operational Dashboards

| Dashboard | Audience | Refresh |
| --- | --- | --- |
| Fleet Health Overview | Operations team | 1 min |
| OTA Deployment Status | Release management | 1 min |
| Security Events | Security team | Real-time |
| Device Registration | Manufacturing ops | 5 min |
| Compliance Report | Compliance officer | Daily |

### 9.2 Scheduled Reports

| Report | Frequency | Audience |
| --- | --- | --- |
| Fleet health summary | Weekly | Product management |
| Security posture report | Monthly | Security architect, CISO |
| Incident summary | Per incident | All stakeholders |
| Compliance evidence package | Per audit cycle | Auditors |

---

## 10. References

| Document | Reference |
| --- | --- |
| Error Code Catalog | `../02_System_Design/Error_Code_Catalog.md` |
| Data Structures | `../02_System_Design/Data_Structures.md` §TelemetryRecord |
| Incident Response | `Incident_Response.md` |
| OTA Deployment | `OTA_Deployment.md` |
| Boot Flow Pseudocode | `../03_Subsystem/Boot/Boot_Flow_Pseudocode.md` |
