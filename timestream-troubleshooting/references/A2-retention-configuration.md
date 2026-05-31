---
title: "A2 — Retention Configuration Issues"
description: "Diagnose Timestream retention policy misconfiguration"
status: active
severity: HIGH
triggers:
  - "retention misconfigured"
  - "data disappearing"
  - "data not expiring"
  - "retention policy"
owner: devops-agent
objective: "Identify and fix retention configuration issues"
context: "Timestream has separate retention for memory store (hours) and magnetic store (days). Data transitions automatically from memory to magnetic. Incorrect retention causes premature data loss or excessive storage costs."
---

## Phase 1 — Triage

MUST:
- Check current retention settings: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.RetentionProperties'`
- Verify memory store retention is in HOURS (1-8760)
- Verify magnetic store retention is in DAYS (1-73000)
- Check if magnetic store writes are enabled: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.MagneticStoreWriteProperties'`

SHOULD:
- Calculate total data retention: memory hours + magnetic days
- Check if data is being rejected due to retention window
- Verify the retention meets business requirements

MAY:
- Check CloudWatch for RejectedRecords metrics
- Review data volume in memory vs magnetic store

## Phase 2 — Remediate

MUST:
- Update retention settings:
  `aws timestream-write update-table --database-name <database> --table-name <table> --retention-properties MemoryStoreRetentionPeriodInHours=<hours>,MagneticStoreRetentionPeriodInDays=<days>`
- Enable magnetic store writes if late-arriving data is expected:
  `aws timestream-write update-table --database-name <database> --table-name <table> --magnetic-store-write-properties EnableMagneticStoreWrites=true`

SHOULD:
- Set memory store retention based on query latency requirements
- Set magnetic store retention based on compliance/analysis needs

MAY:
- Use scheduled queries to aggregate data before it expires
- Monitor storage costs with CloudWatch

## Common Issues

- symptoms: "Data disappearing after a few hours"
  diagnosis: "Memory store retention too short and magnetic store retention not set."
  resolution: "Increase memory store retention or ensure magnetic store retention is adequate."

- symptoms: "Late-arriving data rejected"
  diagnosis: "Data timestamp older than memory store retention and magnetic writes disabled."
  resolution: "Enable magnetic store writes on the table."

- symptoms: "Storage costs unexpectedly high"
  diagnosis: "Magnetic store retention set too long or memory store retention too long."
  resolution: "Review and reduce retention periods to match actual requirements."

## Output Format

```yaml
root_cause: "retention_configuration_issue — <specific_cause>"
evidence:
  - type: retention_settings
    content: "<current memory and magnetic retention>"
  - type: magnetic_writes
    content: "<magnetic store writes enabled/disabled>"
  - type: rejected_records
    content: "<rejected records metrics>"
severity: HIGH
mitigation:
  immediate: "Fix retention settings"
  long_term: "Document retention requirements and automate with IaC"
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
