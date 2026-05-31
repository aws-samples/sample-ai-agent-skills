---
title: "A3 — Implicit Deny (Missing Allow)"
description: "Diagnose access denied caused by missing Allow statements rather than explicit Deny"
status: active
severity: MEDIUM
triggers:
  - "implicit deny"
  - "missing permission"
  - "no matching allow"
  - "forgot to add permission"
owner: devops-agent
objective: "Identify which policy layer is missing the required Allow statement and add it with least privilege"
context: "IAM defaults to deny. If no policy explicitly allows an action, it is implicitly denied. This is the most common cause of access denied errors — the principal simply lacks the required permission in their identity policies."
---

## Phase 1 — Triage

MUST:
- Identify the exact action being denied: check CloudTrail or error message for the API call
- List all policies attached to the principal:
  - `aws iam list-attached-user-policies --user-name <user>`
  - `aws iam list-attached-role-policies --role-name <role>`
  - `aws iam list-user-policies --user-name <user>` (inline)
  - `aws iam list-role-policies --role-name <role>` (inline)
- Check group memberships and group policies: `aws iam list-groups-for-user --user-name <user>`
- Simulate to confirm implicit deny: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action> --resource-arns <resource>`

SHOULD:
- Verify the action name is correct — IAM actions are case-sensitive and service-specific
- Check if the resource ARN format is correct in existing policies (wrong ARN = no match = implicit deny)
- Check for condition keys that prevent the Allow from matching: `aws iam get-policy-version --policy-arn <arn> --version-id <version>`

MAY:
- Use IAM Access Advisor to see which services the principal has accessed: `aws iam generate-service-last-accessed-details --arn <principal-arn>`
- Check AWS managed policies for a suitable pre-built policy: `aws iam list-policies --scope AWS --query "Policies[?contains(PolicyName, '<service>')]"`

## Phase 2 — Remediate

MUST:
- Add the required permission to an identity policy with least privilege
- Specify the exact action(s) needed, not wildcards
- Scope the Resource to the specific ARN(s) needed

SHOULD:
- Prefer managed policies over inline for reusability
- Check that permission boundary (if set) also allows the action
- Test with simulate-principal-policy after adding the permission

MAY:
- Create a custom managed policy for the specific use case
- Use IAM Access Analyzer policy generation to create a least-privilege policy from CloudTrail activity

## Common Issues

- symptoms: "Policy allows s3:GetObject but s3:ListBucket is denied"
  diagnosis: "s3:ListBucket and s3:GetObject are separate actions. ListBucket resource is the bucket ARN, GetObject resource is the object ARN."
  resolution: "Add s3:ListBucket with the bucket ARN as resource."

- symptoms: "Policy allows the action but resource ARN uses * and it's still denied"
  diagnosis: "Some actions require specific resource types. Check if the action supports the resource type."
  resolution: "Verify the IAM action-resource mapping in the IAM documentation. Some actions only work with * as resource."

- symptoms: "Permission was added but the principal is using an assumed role"
  diagnosis: "The permission was added to the user, but the assumed role has different policies."
  resolution: "Add the permission to the role's policies, not the user's."

## Output Format

```yaml
root_cause: "implicit_deny — missing Allow for <action> on <resource>"
evidence:
  - type: simulation_result
    content: "<simulate-principal-policy showing implicitDeny>"
  - type: policy_list
    content: "<list of all policies checked>"
severity: MEDIUM
mitigation:
  immediate: "Add Allow statement for the required action and resource"
  long_term: "Review and consolidate permissions using IAM Access Analyzer"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws iam list-attached-user-policies` / `list-attached-role-policies` | GREEN | Read-only diagnostic |
| `aws iam list-user-policies` / `list-role-policies` | GREEN | Read-only diagnostic |
| `aws iam list-groups-for-user` | GREEN | Read-only diagnostic |
| `aws iam simulate-principal-policy` | GREEN | Read-only diagnostic |
| `aws iam get-policy-version` | GREEN | Read-only diagnostic |
| `aws iam generate-service-last-accessed-details` | GREEN | Read-only diagnostic |
| `aws iam list-policies` | GREEN | Read-only diagnostic |
| `aws iam attach-role-policy` / `attach-user-policy` | YELLOW | Adds permissions to a principal — could widen access |
| `aws iam put-role-policy` / `put-user-policy` | YELLOW | Creates inline policy granting new permissions |
| `aws iam create-policy` with broad permissions | RED | Could grant excessive access if not scoped properly |

## Escalation Conditions

- Remediation requires modifying IAM policies in a production account
- Fix involves granting cross-account access
- Root cause involves the management account
- Multiple principals are affected
- The denied action involves security-sensitive services (KMS, STS, Organizations)

## Rollback

1. **Before any change**: Document current state with `aws iam get-policy-version` or `aws iam get-role`
2. **After change**: Verify the fix resolves the issue without granting excessive permissions
3. **If change causes issues**: Revert to the previous policy version or detach the modified policy
4. **Cleanup**: Remove any temporary policies or test principals created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-account-authorization-details` | HIGH | Contains all IAM entities — redact from stored output |
| `get-policy-version` | HIGH | Contains full policy document — summarize, don't reproduce |
| `simulate-principal-policy` | MEDIUM | Shows effective permissions — summarize results |
| `list-*` | LOW | Resource listings — safe to include |

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting `AdministratorAccess` to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest disabling MFA requirements as a troubleshooting step
- ALWAYS identify the specific denied action from CloudTrail before suggesting a fix
- ALWAYS use least-privilege: grant only the specific action on the specific resource

## Cleanup

- No temporary resources are created during implicit deny analysis

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
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
