---
title: "E2 — Data Cleanup Issues"
description: "Diagnose data cleanup and deletion problems"
status: active
severity: LOW
triggers:
  - "cannot delete data"
  - "old data remaining"
  - "cleanup failed"
owner: devops-agent
objective: "Identify and resolve data cleanup issues"
context: "Cleanup issues occur when: retention not deleting old data, manual deletion fails, or customer-managed S3 lifecycle conflicts."
---

## Phase 1 — Triage

MUST:
- Check retention settings for channel and datastore
- Verify if customer-managed S3 has conflicting lifecycle policies
- Check if data is within retention period (not yet eligible for deletion)

SHOULD:
- Check S3 bucket for orphaned data files
- Verify dataset content retention settings
- Check for failed deletion operations in CloudTrail

MAY:
- Calculate expected data volume vs actual
- Check for reprocessing operations that may have recreated data

## Phase 2 — Remediate

MUST:
- Configure retention periods to enable automatic cleanup
- Align S3 lifecycle policies with IoT Analytics retention
- Manually delete dataset content if needed: `aws iotanalytics delete-dataset-content --dataset-name <name> --version-id <id>`

SHOULD:
- Set up cost monitoring for storage
- Document cleanup procedures

## Output Format

```yaml
root_cause: "cleanup — <specific_cause>"
evidence:
  - type: retention_config
    content: "<retention and lifecycle settings>"
severity: LOW
mitigation:
  immediate: "Configure retention or manual cleanup"
  long_term: "Automate cleanup with retention policies"
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
