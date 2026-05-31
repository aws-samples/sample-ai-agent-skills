---
title: "A1 — Workspace Creation Failures"
description: "Diagnose workspace creation and provisioning failures"
status: active
severity: HIGH
triggers: ["workspace creation failed", "cannot create workspace", "S3 bucket error"]
owner: devops-agent
objective: "Identify and resolve workspace creation failures"
context: "Creation fails when: S3 bucket doesn't exist, IAM role insufficient, bucket in wrong region, or account limits reached."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <bucket-name>`
- Check IAM role: `aws iam get-role --role-name <role-name>`
- Verify role has s3:GetObject, s3:PutObject, s3:DeleteObject on the bucket
- Check CloudTrail for creation errors

SHOULD:
- Verify bucket is in same region as workspace
- Check role trust policy allows iottwinmaker.amazonaws.com
- Verify account limits for workspaces

## Phase 2 — Remediate

MUST:
- Create S3 bucket in correct region
- Fix IAM role permissions and trust policy
- Ensure bucket name is unique

SHOULD:
- Use dedicated bucket per workspace
- Enable bucket versioning for content protection

## Output Format

```yaml
root_cause: "workspace_creation — <specific_cause>"
evidence:
  - type: iam_role
    content: "<role permissions>"
severity: HIGH
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
- Never delete workspaces containing active digital twin entities
- Never modify component types that are referenced by production entities
- Never remove data connectors without confirming alternative data sources
- Never alter scene configurations used by active monitoring dashboards
- Never delete entity hierarchies without understanding downstream visualization impact

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous component type definitions if entity data binding breaks
- Revert scene configuration changes if 3D visualization stops rendering
- Re-enable previous data connector settings if twin data synchronization fails

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
