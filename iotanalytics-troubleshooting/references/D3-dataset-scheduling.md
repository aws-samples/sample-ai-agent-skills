---
title: "D3 — Dataset Scheduling Issues"
description: "Diagnose dataset schedule and trigger problems"
status: active
severity: MEDIUM
triggers:
  - "dataset not running"
  - "schedule not triggering"
  - "cron expression error"
owner: devops-agent
objective: "Identify and resolve dataset scheduling issues"
context: "Scheduling issues occur when: cron expression invalid, trigger dataset doesn't exist, or overlapping executions queue up."
---

## Phase 1 — Triage

MUST:
- Check dataset schedule configuration: `aws iotanalytics describe-dataset --dataset-name <name>`
- Verify cron expression syntax
- Check dataset content history for execution status
- Verify trigger dataset exists if using dataset trigger

SHOULD:
- Check for overlapping executions
- Verify dataset is not in error state
- Check CloudWatch for dataset execution metrics

MAY:
- Manually trigger dataset: `aws iotanalytics create-dataset-content --dataset-name <name>`
- Check execution history for patterns

## Phase 2 — Remediate

MUST:
- Fix cron expression syntax
- Fix trigger dataset reference
- Clear error state if dataset stuck

SHOULD:
- Set appropriate schedule intervals
- Monitor execution success rates

## Output Format

```yaml
root_cause: "dataset_scheduling — <specific_cause>"
evidence:
  - type: schedule_config
    content: "<schedule details>"
severity: MEDIUM
mitigation:
  immediate: "Fix schedule or trigger"
  long_term: "Monitor dataset execution health"
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
