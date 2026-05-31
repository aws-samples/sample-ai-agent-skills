---
title: "A2 — Channel Configuration Issues"
description: "Diagnose channel storage and retention configuration problems"
status: active
severity: MEDIUM
triggers:
  - "channel storage full"
  - "retention not working"
  - "S3 storage error"
owner: devops-agent
objective: "Identify and resolve channel configuration issues"
context: "Configuration issues occur when: customer-managed S3 permissions wrong, retention period misconfigured, or storage type incompatible."
---

## Phase 1 — Triage

MUST:
- Check channel configuration: `aws iotanalytics describe-channel --channel-name <name>`
- Verify storage type (service-managed vs customer-managed S3)
- Check retention period settings
- If customer-managed S3: verify bucket permissions and IAM role

SHOULD:
- Check S3 bucket policy for customer-managed storage
- Verify KMS key permissions if encryption configured
- Check channel status for errors

MAY:
- Check S3 bucket for data files
- Review CloudTrail for channel API errors

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions for customer-managed storage
- Configure appropriate retention period
- Fix KMS key permissions if using encryption

SHOULD:
- Set retention to manage storage costs
- Monitor storage usage

## Output Format

```yaml
root_cause: "channel_config — <specific_cause>"
evidence:
  - type: channel_details
    content: "<channel configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix storage permissions"
  long_term: "Configure retention policies"
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
