---
title: "C1 — Policy Size Limits"
description: "Diagnose and resolve policy size limit errors for managed, inline, and trust policies"
status: active
severity: MEDIUM
triggers:
  - "policy size"
  - "character limit"
  - "MalformedPolicyDocument"
  - "policy too large"
  - "LimitExceeded"
owner: devops-agent
objective: "Identify which policy size limit is being hit and restructure policies to fit within limits"
context: "IAM enforces character limits on policies: managed policies (6,144 chars), inline policies (2,048 for users, 10,240 for roles/groups), trust policies (2,048 chars for roles, 4,096 with extended limit). Whitespace counts. These limits cannot be increased."
---

## Phase 1 — Triage

MUST:
- Identify the error: look for `MalformedPolicyDocument` or `LimitExceeded` in the error message
- Check current policy size: `aws iam get-policy-version --policy-arn <arn> --version-id <version>` and count characters
- Identify the policy type (managed, inline, trust) to determine the applicable limit

SHOULD:
- List all policies on the principal to see total policy count:
  - `aws iam list-attached-user-policies --user-name <user>`
  - `aws iam list-user-policies --user-name <user>`
- Check if whitespace/formatting is inflating the policy size
- Identify redundant or overlapping statements

MAY:
- Use `aws iam get-account-authorization-details` to audit all policies in the account
- Check if AWS managed policies can replace custom policies

## Phase 2 — Remediate

MUST:
- Reduce policy size by:
  - Using wildcards in actions where appropriate (e.g., `s3:Get*` instead of listing each Get action)
  - Consolidating resource ARNs with wildcards
  - Removing redundant statements
  - Removing whitespace (IAM ignores formatting)
- If a single policy cannot fit, split into multiple policies

SHOULD:
- Use managed policies (6,144 char limit) instead of inline user policies (2,048 char limit)
- Combine multiple inline policies into a single managed policy
- Use policy variables (e.g., `${aws:username}`) to reduce resource ARN duplication

MAY:
- Use permission boundaries to reduce the need for explicit Deny statements
- Use tag-based access control (ABAC) to reduce the number of resource ARNs

## Common Issues

- symptoms: "Cannot create inline policy on user — LimitExceeded"
  diagnosis: "User inline policies are limited to 2,048 characters."
  resolution: "Convert to a managed policy (6,144 char limit) or attach to a group."

- symptoms: "Trust policy update fails with MalformedPolicyDocument"
  diagnosis: "Trust policies are limited to 2,048 characters (4,096 with extended limit)."
  resolution: "Reduce the number of principals. Use aws:PrincipalOrgID condition instead of listing individual accounts."

- symptoms: "Policy fits in the console but fails via API"
  diagnosis: "The console may strip whitespace. The API counts all characters including whitespace."
  resolution: "Minify the policy JSON before submitting via API."

## Output Format

```yaml
root_cause: "policy_size — <policy_type>: <current_size>/<limit> characters"
evidence:
  - type: policy_document
    content: "<policy with size annotation>"
  - type: error_message
    content: "<MalformedPolicyDocument or LimitExceeded>"
severity: MEDIUM
mitigation:
  immediate: "Reduce policy size or split into multiple policies"
  long_term: "Adopt ABAC and managed policies to minimize policy size"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws iam get-policy-version` | GREEN | Read-only diagnostic |
| `aws iam list-attached-user-policies` / `list-user-policies` | GREEN | Read-only diagnostic |
| `aws iam get-account-authorization-details` | GREEN | Read-only diagnostic |
| `aws iam create-policy-version --set-as-default` (restructured policy) | YELLOW | Changing policy content can inadvertently alter effective permissions |
| `aws iam put-user-policy` / `put-role-policy` (split inline) | YELLOW | Creating new inline policies changes effective permissions |
| `aws iam attach-role-policy` (new managed policy) | YELLOW | Attaching new policies changes effective permissions |

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

- If policies were split into multiple new managed policies during restructuring, ensure orphaned or unused intermediate policy versions are deleted

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
