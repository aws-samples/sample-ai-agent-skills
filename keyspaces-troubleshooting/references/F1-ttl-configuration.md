---
title: "F1 — TTL Configuration"
description: "Diagnose Amazon Keyspaces TTL configuration issues"
status: active
severity: MEDIUM
triggers:
  - "TTL configuration"
  - "time to live setup"
  - "TTL not enabled"
  - "TTL value rejected"
  - "default TTL"
owner: devops-agent
objective: "Resolve TTL configuration issues in Amazon Keyspaces"
context: "Keyspaces TTL must be enabled on the table before use. TTL values range from 1 to 630,720,000 seconds (20 years). TTL can be set per-row via USING TTL clause or as a default table-level TTL. TTL is enabled via the table's ttl property. Expired data deletion consumes write capacity."
---

## Phase 1 — Triage

MUST:
- Check if TTL is enabled on the table: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'ttl'`
- Verify the TTL value is within valid range (1 to 630,720,000 seconds)
- Check if the table has a default TTL configured
- Verify the INSERT/UPDATE statement uses correct USING TTL syntax

SHOULD:
- Check write capacity for TTL deletion overhead
- Verify application is setting TTL values correctly on writes
- Review CloudTrail for TTL configuration changes

MAY:
- Estimate write capacity impact of TTL expirations
- Check if TTL is being set on the correct column

## Phase 2 — Remediate

MUST:
- Enable TTL on the table: `aws keyspaces update-table --keyspace-name <keyspace> --table-name <table> --ttl status=ENABLED`
- Use valid TTL values between 1 and 630,720,000 seconds
- Set TTL in CQL: `INSERT INTO keyspace.table (pk, col) VALUES ('key', 'val') USING TTL 86400;`

SHOULD:
- Set default table TTL for uniform expiration: configure via table properties
- Account for write capacity consumed by TTL deletions
- Monitor TTL expiration patterns

MAY:
- Implement TTL management in application logic
- Use different TTL values for different data categories

## Common Issues

- symptoms: "TTL value rejected — exceeds maximum"
  diagnosis: "TTL value exceeds 630,720,000 seconds (20 years)."
  resolution: "Use a TTL value within the valid range. Maximum is 630,720,000 seconds."

- symptoms: "USING TTL clause ignored — TTL not enabled on table"
  diagnosis: "TTL feature must be enabled on the table before per-row TTL works."
  resolution: "Enable TTL: aws keyspaces update-table --ttl status=ENABLED"

- symptoms: "Unexpected write capacity consumption"
  diagnosis: "TTL expirations consume write capacity units when deleting expired rows."
  resolution: "Account for TTL deletion WCU in capacity planning. Monitor ConsumedWriteCapacityUnits."

## Output Format

```yaml
root_cause: "ttl_configuration — <specific_cause>"
evidence:
  - type: ttl_status
    content: "<TTL enabled/disabled on table>"
  - type: ttl_value
    content: "<TTL value used and validity>"
  - type: capacity_impact
    content: "<write capacity consumed by TTL deletions>"
severity: MEDIUM
mitigation:
  immediate: "Enable TTL and use valid TTL values"
  long_term: "Plan capacity for TTL deletion overhead"
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
