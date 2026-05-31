---
title: "C1 — Data Store Issues"
description: "Diagnose data store storage and access problems"
status: active
severity: MEDIUM
triggers:
  - "datastore empty"
  - "data not queryable"
  - "storage error"
owner: devops-agent
objective: "Identify and resolve data store issues"
context: "Data store issues occur when: pipeline not writing, S3 permissions wrong, retention deleting data, or partition configuration invalid."
---

## Phase 1 — Triage

MUST:
- Check datastore status: `aws iotanalytics describe-datastore --datastore-name <name>`
- Verify pipeline is writing to this datastore
- Check retention period settings
- If customer-managed S3: verify bucket permissions

SHOULD:
- Check datastore statistics for data volume
- Verify partition configuration
- Check if retention is deleting expected data

MAY:
- Check S3 bucket directly for data files
- Verify file format (JSON or Parquet)

## Phase 2 — Remediate

MUST:
- Fix pipeline to write to correct datastore
- Fix S3 permissions for customer-managed storage
- Adjust retention period if data being deleted prematurely

SHOULD:
- Configure appropriate partitioning for query patterns
- Monitor datastore size and growth

## Output Format

```yaml
root_cause: "datastore_issue — <specific_cause>"
evidence:
  - type: datastore_config
    content: "<datastore details>"
severity: MEDIUM
mitigation:
  immediate: "Fix pipeline or storage permissions"
  long_term: "Optimize partitioning and retention"
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
