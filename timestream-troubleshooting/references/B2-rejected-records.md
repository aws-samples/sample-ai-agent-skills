---
title: "B2 — Rejected Records"
description: "Diagnose Timestream rejected records errors"
status: active
severity: HIGH
triggers:
  - "rejected records"
  - "RejectedRecordsException"
  - "records not accepted"
  - "data rejected"
owner: devops-agent
objective: "Identify why records are being rejected and fix the issue"
context: "Records are rejected for late-arriving data outside the memory store window (when magnetic writes disabled), duplicate records, invalid measure types, or exceeding dimension limits. The RejectedRecordsException includes details about which records failed."
---

## Phase 1 — Triage

MUST:
- Check the RejectedRecordsException details in application logs for specific rejection reasons
- Verify memory store retention: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.RetentionProperties.MemoryStoreRetentionPeriodInHours'`
- Check if magnetic store writes are enabled: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.MagneticStoreWriteProperties'`
- Check CloudWatch RejectedRecords metric: `aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name UserErrors --dimensions Name=DatabaseName,Value=<database> Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Verify record timestamps are within the memory store retention window
- Check for measure type mismatches (e.g., sending string for a DOUBLE measure)
- Verify dimension count is within limits

MAY:
- Analyze the pattern of rejected records (time-based, dimension-based)
- Check if the data source is sending stale data

## Phase 2 — Remediate

MUST:
- Enable magnetic store writes for late-arriving data:
  `aws timestream-write update-table --database-name <database> --table-name <table> --magnetic-store-write-properties EnableMagneticStoreWrites=true`
- Fix record timestamps to be within the retention window
- Fix measure type mismatches in the data pipeline

SHOULD:
- Implement record validation before sending to Timestream
- Handle RejectedRecordsException in application code (partial batch success)

MAY:
- Increase memory store retention if data arrives with slight delays
- Set up alerting for rejected records

## Common Issues

- symptoms: "RejectedRecordsException with reason 'Data is outside retention range'"
  diagnosis: "Record timestamp older than memory store retention and magnetic writes disabled."
  resolution: "Enable magnetic store writes or increase memory store retention."

- symptoms: "RejectedRecordsException with measure type mismatch"
  diagnosis: "Sending a different measure value type than previously written for the same measure name."
  resolution: "Ensure consistent measure types. Once established, the type cannot change."

- symptoms: "Partial batch rejection"
  diagnosis: "Some records in the batch are valid, others are not."
  resolution: "Handle RejectedRecordsException — it includes which records failed. Retry only failed records after fixing."

## Output Format

```yaml
root_cause: "rejected_records — <specific_cause>"
evidence:
  - type: rejection_reason
    content: "<specific rejection reason from exception>"
  - type: retention_config
    content: "<memory and magnetic retention settings>"
  - type: magnetic_writes
    content: "<magnetic store writes status>"
severity: HIGH
mitigation:
  immediate: "Fix the rejection cause (enable magnetic writes, fix types)"
  long_term: "Implement record validation and monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
