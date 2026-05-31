---
title: "E1 — IAM Permissions"
description: "Diagnose IAM permission issues for Resource Explorer"
status: active
severity: HIGH
triggers: ["access denied", "permission error", "unauthorized", "cannot search", "cannot create index"]
owner: devops-agent
objective: "Resolve IAM permission issues for Resource Explorer operations"
context: "Resource Explorer has separate permissions for search, index management, and view management."
---

## Phase 1 — Triage

MUST:
- Verify caller identity: `aws sts get-caller-identity`
- Test specific permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names resource-explorer-2:Search resource-explorer-2:CreateIndex resource-explorer-2:ListViews`
- Check for explicit denies in IAM policies or SCPs

SHOULD:
- Review the IAM role/user policies: `aws iam list-attached-user-policies --user-name <name>` or `aws iam list-attached-role-policies --role-name <name>`
- Check for permission boundaries

MAY:
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Search --max-results 10`
- Check SCPs in the organization

## Phase 2 — Remediate

MUST:
- Grant required permissions based on the user's role:
  - Search users: resource-explorer-2:Search, resource-explorer-2:GetView, resource-explorer-2:ListViews
  - Index admins: resource-explorer-2:CreateIndex, resource-explorer-2:DeleteIndex, resource-explorer-2:UpdateIndexType
  - View admins: resource-explorer-2:CreateView, resource-explorer-2:UpdateView, resource-explorer-2:DeleteView
- Remove explicit denies that block required actions

SHOULD:
- Apply least-privilege principle
- Use IAM policy conditions for additional security

MAY:
- Create custom IAM policies for different Resource Explorer roles
- Set up IAM Access Analyzer to monitor permission usage

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "iam list-attached-user-policies: GREEN — read-only policy listing"
  - "iam attach-user-policy: YELLOW — modifies IAM permissions"
```

## Escalation Conditions

- SCPs block Resource Explorer and require organization admin changes
- Permission boundaries prevent granting required permissions
- Cross-account access requires trust policy modifications
- IAM permission changes affect other services or users

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — reveals account ID and principal
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws iam list-attached-user-policies`: MEDIUM — reveals IAM policy attachments
- IAM policy content: HIGH — reveals permission boundaries

## Prohibited Actions

- NEVER grant resource-explorer-2:* as a quick fix
- NEVER modify IAM policies without understanding the blast radius
- NEVER remove permission boundaries to fix access issues
- NEVER share IAM policy details without reviewing for sensitive information

## Phase 3 — Rollback

1. If IAM policies were modified, restore from backup or version control
2. If a managed policy was attached, detach it: `aws iam detach-user-policy --user-name <name> --policy-arn <arn>`
3. Verify the user can still perform their required operations
4. Check that no excessive permissions remain after troubleshooting
5. Audit IAM changes with CloudTrail

## Common Issues

- symptoms: "Search returns AccessDeniedException"
  diagnosis: "The caller lacks resource-explorer-2:Search permission or an SCP denies it."
  resolution: "Grant resource-explorer-2:Search and resource-explorer-2:GetView permissions. Check SCPs for explicit denies."

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: permission_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Create role-based IAM policies for Resource Explorer"
```

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
