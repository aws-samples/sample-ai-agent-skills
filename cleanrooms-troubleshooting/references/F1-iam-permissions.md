---
title: "F1 — IAM Permission Issues"
description: "Diagnose Clean Rooms IAM permission problems including service roles and cross-account access"
status: active
severity: HIGH
triggers:
  - "clean rooms permissions"
  - "access denied clean rooms"
  - "iam clean rooms"
  - "service role"
  - "permission error"
owner: devops-agent
objective: "Resolve IAM permission issues for Clean Rooms operations"
context: "Clean Rooms requires IAM permissions for API operations and service roles for data access. Service roles need S3 and Glue permissions. Common issues include missing service role, insufficient permissions, and cross-account role configuration."
---

## Phase 1 — Triage

MUST:
- Verify caller identity: `aws sts get-caller-identity`
- Check IAM permissions for Clean Rooms actions (`cleanrooms:*`)
- Verify service role exists and has correct trust policy
- Check service role has S3 and Glue permissions for data access
- Check CloudTrail for access denied events

SHOULD:
- Verify service role trust policy allows `cleanrooms.amazonaws.com`
- Check for SCP restrictions on Clean Rooms operations
- Verify separate permissions for `cleanrooms-ml` if using ML features

MAY:
- Use IAM Policy Simulator to test permissions
- Check for permission boundaries

## Phase 2 — Remediate

MUST:
- Create service role with correct trust policy and permissions
- Grant `cleanrooms:*` to the user/role performing operations
- Ensure service role has `s3:GetObject`, `s3:ListBucket`, `glue:GetTable`, `glue:GetDatabase`

SHOULD:
- Use managed policies where available
- Separate admin and query permissions

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: caller_identity
    content: "<sts get-caller-identity>"
  - type: service_role
    content: "<service role configuration>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or service role"
  long_term: "Use managed policies and implement least-privilege"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create service role with correct trust policy | YELLOW |
| Grant cleanrooms:* to user/role | YELLOW |
| Ensure service role has S3 and Glue permissions | YELLOW |
| Use managed policies | GREEN |
| Separate admin and query permissions | GREEN |

## Escalation Conditions

- Fix requires modifying production IAM roles or policies
- Fix involves granting broad Clean Rooms permissions

## Data Sensitivity

- **HIGH**: IAM configurations control access to all Clean Rooms operations and data
- Service role permissions determine the scope of data accessible through collaborations

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If IAM permissions were broadened, revert to the previous policy version
2. If service role was created, delete it if no longer needed
3. If trust policy was modified, restore previous trust configuration
4. Verify rollback by confirming IAM policies match expected least-privilege configuration

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
