---
title: "E1 — Data Retention Issues"
description: "Diagnose data retention and lifecycle problems"
status: active
severity: MEDIUM
triggers:
  - "data disappeared"
  - "retention deleting data"
  - "data not expiring"
owner: devops-agent
objective: "Identify and resolve data retention issues"
context: "Retention issues occur when: retention period too short, retention not configured, or channel vs datastore retention confused."
---

## Phase 1 — Triage

MUST:
- Check channel retention: `aws iotanalytics describe-channel --channel-name <name>`
- Check datastore retention: `aws iotanalytics describe-datastore --datastore-name <name>`
- Check dataset content retention in dataset configuration
- Verify which retention period is causing data loss

SHOULD:
- Distinguish between channel retention (raw data) and datastore retention (processed data)
- Check if unlimited retention is causing storage cost issues
- Verify retention period units (days)

MAY:
- Check S3 lifecycle policies for customer-managed storage
- Review storage costs in Cost Explorer

## Phase 2 — Remediate

MUST:
- Set appropriate retention periods for channels, datastores, and datasets
- Extend retention if data being deleted prematurely
- Reduce retention if storage costs too high

SHOULD:
- Align retention periods across channels and datastores
- Document retention policies

## Output Format

```yaml
root_cause: "data_retention — <specific_cause>"
evidence:
  - type: retention_config
    content: "<retention settings>"
severity: MEDIUM
mitigation:
  immediate: "Adjust retention periods"
  long_term: "Implement retention policy governance"
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
