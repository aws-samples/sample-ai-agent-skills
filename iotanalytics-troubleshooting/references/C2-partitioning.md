---
title: "C2 — Data Store Partitioning Issues"
description: "Diagnose data store partitioning and query performance"
status: active
severity: MEDIUM
triggers:
  - "slow queries"
  - "partition error"
  - "full scan"
owner: devops-agent
objective: "Identify and resolve data store partitioning issues"
context: "Partitioning issues occur when: no custom partitions defined, partition keys don't match query patterns, or __dt not used in filters."
---

## Phase 1 — Triage

MUST:
- Check datastore partition configuration: `aws iotanalytics describe-datastore --datastore-name <name>`
- Review dataset SQL queries for partition filter usage
- Check if __dt column is used in WHERE clauses
- Verify file format (JSON vs Parquet)

SHOULD:
- Check query execution times
- Verify partition keys match common query patterns
- Check data volume and growth rate

MAY:
- Compare query performance with and without partition filters
- Check S3 prefix structure for customer-managed storage

## Phase 2 — Remediate

MUST:
- Add __dt filter to all time-based queries
- Use Parquet format for better query performance
- Note: custom partitions cannot be changed after creation

SHOULD:
- Create new datastore with better partitioning if needed
- Optimize queries to leverage existing partitions

## Output Format

```yaml
root_cause: "partitioning — <specific_cause>"
evidence:
  - type: partition_config
    content: "<partition details>"
severity: MEDIUM
mitigation:
  immediate: "Add partition filters to queries"
  long_term: "Redesign partitioning strategy"
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
- Never delete channels or datastores that are actively receiving data
- Never modify pipeline activities on production pipelines without testing
- Never change data retention settings without understanding downstream impact
- Never delete datasets that feed dashboards or reporting systems
- Never alter SQL query definitions in scheduled datasets without validation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous pipeline activity configuration if data processing breaks
- Revert channel or datastore settings if data ingestion stops
- Re-enable previous dataset scheduling if automated analysis is disrupted

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
