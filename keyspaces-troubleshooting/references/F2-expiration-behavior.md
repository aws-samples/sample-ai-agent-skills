---
title: "F2 — TTL Expiration Behavior"
description: "Diagnose Amazon Keyspaces TTL expiration behavior issues"
status: active
severity: MEDIUM
triggers:
  - "TTL not expiring"
  - "expired data still visible"
  - "data not deleted"
  - "TTL delay"
  - "stale data after TTL"
owner: devops-agent
objective: "Explain and resolve TTL expiration behavior issues"
context: "Keyspaces TTL expiration is asynchronous. Expired rows may be visible briefly after the TTL timestamp passes. TTL deletions happen in the background and consume write capacity. Setting TTL to 0 removes the TTL from a row. Row-level TTL overrides table-level default TTL."
---

## Phase 1 — Triage

MUST:
- Verify TTL is enabled on the table: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'ttl'`
- Check the TTL value set on the specific row (query with TTL function if available)
- Verify the row's write timestamp and TTL to calculate expected expiration time
- Check write capacity metrics for TTL deletion activity: `aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Confirm the TTL was set correctly during INSERT/UPDATE
- Check if row-level TTL overrides table default TTL
- Verify system clock alignment (TTL uses server-side timestamps)

MAY:
- Monitor the delay between TTL expiration and actual deletion
- Check if high write load is delaying TTL background deletions

## Phase 2 — Remediate

MUST:
- Understand that TTL expiration is asynchronous — brief visibility after expiration is expected
- Filter expired data in application queries if immediate consistency is required
- Ensure write capacity is sufficient for TTL deletion overhead

SHOULD:
- Implement application-level filtering for time-sensitive data
- Monitor TTL deletion patterns and adjust capacity accordingly
- Use appropriate TTL values (not excessively long)

MAY:
- Implement a cleanup job for data that must be removed immediately
- Set up alerts for TTL deletion capacity consumption

## Common Issues

- symptoms: "Expired rows still returned in queries"
  diagnosis: "TTL expiration is asynchronous. Brief visibility after expiration is normal."
  resolution: "This is expected behavior. Filter expired data in application if needed."

- symptoms: "TTL set to 0 but data not expiring"
  diagnosis: "TTL value of 0 removes the TTL from the row (makes it permanent)."
  resolution: "Use TTL value of 1 (minimum) for near-immediate expiration, not 0."

- symptoms: "Table default TTL not applying to new rows"
  diagnosis: "Row-level TTL overrides table default. Rows inserted with explicit TTL 0 are permanent."
  resolution: "Do not set TTL 0 on INSERT if you want the table default to apply. Omit the USING TTL clause."

## Output Format

```yaml
root_cause: "ttl_expiration — <specific_cause>"
evidence:
  - type: ttl_config
    content: "<table and row TTL settings>"
  - type: expiration_timing
    content: "<expected vs actual expiration>"
  - type: deletion_capacity
    content: "<write capacity consumed by deletions>"
severity: MEDIUM
mitigation:
  immediate: "Filter expired data in application if immediate removal needed"
  long_term: "Plan capacity for TTL deletions and implement monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
