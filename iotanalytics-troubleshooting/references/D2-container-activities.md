---
title: "D2 — Container Dataset Issues"
description: "Diagnose container dataset execution failures"
status: active
severity: MEDIUM
triggers:
  - "container failed"
  - "Docker error"
  - "container timeout"
  - "ECR access denied"
owner: devops-agent
objective: "Identify and resolve container dataset execution failures"
context: "Container failures occur when: ECR image not found, execution role lacks permissions, container crashes, or S3 I/O fails."
---

## Phase 1 — Triage

MUST:
- Check dataset definition for container configuration
- Verify ECR image exists: `aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>`
- Check execution role permissions for S3 and ECR
- Check dataset content status for error messages

SHOULD:
- Verify container can read input from S3 path
- Check container output S3 path is writable
- Review container logs if available

MAY:
- Test container locally with sample data
- Check ECS task execution logs

## Phase 2 — Remediate

MUST:
- Fix ECR image reference
- Grant execution role ecr:GetDownloadUrlForLayer, s3:GetObject, s3:PutObject
- Fix container code errors

SHOULD:
- Set appropriate container timeout
- Implement error handling in container code

## Output Format

```yaml
root_cause: "container_dataset — <specific_cause>"
evidence:
  - type: container_status
    content: "<execution error>"
severity: MEDIUM
mitigation:
  immediate: "Fix image or permissions"
  long_term: "Implement container testing pipeline"
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
