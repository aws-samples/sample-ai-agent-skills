---
title: "D1 — Memory Store Issues"
description: "Diagnose Timestream memory store configuration and behavior problems"
status: active
severity: MEDIUM
triggers:
  - "memory store"
  - "hot data"
  - "recent data query"
  - "memory retention"
owner: devops-agent
objective: "Identify and resolve memory store configuration issues"
context: "Memory store holds recent data for low-latency queries. Retention is 1-8760 hours. Data automatically transitions to magnetic store when it ages past the retention window. Memory store is more expensive but faster."
---

## Phase 1 — Triage

MUST:
- Check memory store retention: `aws timestream-write describe-table --database-name <database> --table-name <table> --query 'Table.RetentionProperties.MemoryStoreRetentionPeriodInHours'`
- Verify data is being written within the retention window
- Check query performance for recent data
- Verify the retention period meets requirements

SHOULD:
- Compare query latency for memory vs magnetic store data
- Check write patterns to ensure data arrives within the retention window
- Monitor memory store utilization

MAY:
- Analyze cost implications of memory store retention settings
- Check if retention can be reduced to save costs

## Phase 2 — Remediate

MUST:
- Adjust memory store retention to match requirements:
  `aws timestream-write update-table --database-name <database> --table-name <table> --retention-properties MemoryStoreRetentionPeriodInHours=<hours>,MagneticStoreRetentionPeriodInDays=<days>`
- Ensure data pipeline writes within the retention window

SHOULD:
- Balance retention between cost and query performance
- Use memory store for data that needs sub-second query latency

MAY:
- Set up cost monitoring for memory store usage
- Create alerts for data arriving outside the retention window

## Common Issues

- symptoms: "Recent data queries slower than expected"
  diagnosis: "Data already transitioned to magnetic store due to short retention."
  resolution: "Increase memory store retention to keep recent data in memory longer."

- symptoms: "Memory store costs too high"
  diagnosis: "Retention period longer than needed."
  resolution: "Reduce memory store retention. Data moves to cheaper magnetic store."

- symptoms: "Data written but not queryable immediately"
  diagnosis: "Eventual consistency or endpoint caching issue."
  resolution: "Data should be queryable within seconds. Check endpoint discovery and retry."

## Output Format

```yaml
root_cause: "memory_store_issue — <specific_cause>"
evidence:
  - type: retention_config
    content: "<memory store retention hours>"
  - type: query_latency
    content: "<query performance metrics>"
  - type: write_patterns
    content: "<data arrival timing>"
severity: MEDIUM
mitigation:
  immediate: "Adjust memory store retention"
  long_term: "Optimize retention based on query patterns and cost"
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
