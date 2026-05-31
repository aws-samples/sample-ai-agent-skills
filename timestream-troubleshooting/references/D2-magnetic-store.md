---
title: "D2 — Magnetic Store Issues"
description: "Diagnose Timestream magnetic store configuration and write problems"
status: active
severity: MEDIUM
triggers:
  - "magnetic store"
  - "cold data"
  - "magnetic writes"
  - "historical data"
owner: devops-agent
objective: "Identify and resolve magnetic store configuration issues"
context: "Magnetic store holds historical data at lower cost. Retention is 1-73000 days. Magnetic store writes must be explicitly enabled for late-arriving data. Queries on magnetic store have higher latency than memory store."
---

## Phase 1 — Triage

MUST:
- Check magnetic store retention: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.RetentionProperties.MagneticStoreRetentionPeriodInDays'`
- Check if magnetic store writes are enabled: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.MagneticStoreWriteProperties'`
- Verify data is transitioning from memory to magnetic correctly
- Check for rejected records due to magnetic writes being disabled

SHOULD:
- Check magnetic store write rejected records S3 location
- Verify query performance on magnetic store data
- Monitor storage costs

MAY:
- Analyze data volume in magnetic store
- Check if scheduled queries can pre-aggregate magnetic data

## Phase 2 — Remediate

MUST:
- Enable magnetic store writes if needed:
  `aws timestream-write update-table --database-name <database> --table-name <table> --magnetic-store-write-properties EnableMagneticStoreWrites=true`
- Configure S3 location for rejected magnetic store writes
- Adjust magnetic store retention as needed

SHOULD:
- Use scheduled queries to aggregate magnetic store data for faster queries
- Monitor magnetic store costs

MAY:
- Set up lifecycle policies for magnetic store data
- Create dashboards for storage utilization

## Common Issues

- symptoms: "Late-arriving data rejected"
  diagnosis: "Magnetic store writes not enabled on the table."
  resolution: "Enable magnetic store writes: EnableMagneticStoreWrites=true."

- symptoms: "Historical queries very slow"
  diagnosis: "Querying large amounts of magnetic store data."
  resolution: "Use scheduled queries to pre-aggregate. Add time and dimension filters."

- symptoms: "Data not appearing in magnetic store after memory retention expires"
  diagnosis: "Data transitions automatically — check if the query time range is correct."
  resolution: "Verify the query covers the correct time range for magnetic store data."

## Output Format

```yaml
root_cause: "magnetic_store_issue — <specific_cause>"
evidence:
  - type: magnetic_config
    content: "<magnetic store retention and write settings>"
  - type: rejected_records
    content: "<rejected records due to magnetic writes>"
  - type: query_performance
    content: "<magnetic store query latency>"
severity: MEDIUM
mitigation:
  immediate: "Enable magnetic store writes or fix configuration"
  long_term: "Optimize magnetic store usage with scheduled queries"
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
