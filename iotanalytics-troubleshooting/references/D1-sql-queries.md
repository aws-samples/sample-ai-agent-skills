---
title: "D1 — SQL Dataset Query Issues"
description: "Diagnose SQL dataset query failures"
status: active
severity: MEDIUM
triggers:
  - "SQL error"
  - "query failed"
  - "dataset empty"
  - "no results"
owner: devops-agent
objective: "Identify and resolve SQL dataset query failures"
context: "SQL query issues occur when: syntax error (Presto SQL), datastore name wrong, time filter incorrect, or column names don't match."
---

## Phase 1 — Triage

MUST:
- Check dataset definition: `aws iotanalytics describe-dataset --dataset-name <name>`
- Review SQL query syntax (Presto SQL dialect)
- Verify datastore name in FROM clause
- Check dataset content status: `aws iotanalytics get-dataset-content --dataset-name <name>`

SHOULD:
- Verify column names match datastore schema
- Check __dt partition filter for time-based queries
- Verify query doesn't exceed timeout limits

MAY:
- Test with simplified query first
- Check datastore has data in the queried time range

## Phase 2 — Remediate

MUST:
- Fix SQL syntax to use Presto dialect
- Correct datastore name and column references
- Add __dt filter for efficient time-based queries

SHOULD:
- Test queries with LIMIT before running full queries
- Optimize queries for large data volumes

## Output Format

```yaml
root_cause: "sql_query — <specific_cause>"
evidence:
  - type: dataset_status
    content: "<query error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix SQL syntax"
  long_term: "Optimize queries and add monitoring"
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
