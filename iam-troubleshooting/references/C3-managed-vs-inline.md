---
title: "C3 — Managed vs Inline Policies"
description: "Guidance on when to use managed vs inline policies, their limits, and migration"
status: active
severity: MEDIUM
triggers:
  - "managed policy"
  - "inline policy"
  - "policy type"
  - "which policy type"
  - "too many policies"
owner: devops-agent
objective: "Determine the correct policy type for the use case and resolve policy attachment limit issues"
context: "Managed policies are standalone, reusable, versioned, and have a 6,144 char limit. Inline policies are embedded in a principal, not reusable, and have lower char limits (2,048 for users, 10,240 for roles/groups). Each principal can have up to 10 managed policies attached and unlimited inline policies (within total size limits)."
---

## Phase 1 — Triage

MUST:
- List managed policies: `aws iam list-attached-user-policies --user-name <user>` or `aws iam list-attached-role-policies --role-name <role>`
- List inline policies: `aws iam list-user-policies --user-name <user>` or `aws iam list-role-policies --role-name <role>`
- Count total policies and check against limits (10 managed per principal)

SHOULD:
- Check if duplicate permissions exist across managed and inline policies
- Verify the correct policy type is being used for the use case
- Check total inline policy size against limits

MAY:
- Audit all policies in the account: `aws iam get-account-authorization-details`
- Identify AWS managed policies that could replace custom policies

## Phase 2 — Remediate

MUST:
- Use managed policies for: shared permissions across multiple principals, standard permission sets, policies that need versioning
- Use inline policies for: strict 1:1 relationship with a principal, policies that must be deleted when the principal is deleted
- If hitting the 10 managed policy limit, consolidate policies or use inline for principal-specific permissions

SHOULD:
- Prefer customer managed policies over inline for most use cases
- Use AWS managed policies where they fit (e.g., ReadOnlyAccess, PowerUserAccess)
- Migrate inline policies to managed when they need to be shared

MAY:
- Use permission boundaries to reduce the number of explicit Deny policies needed
- Implement ABAC to reduce the total number of policies

## Common Issues

- symptoms: "Cannot attach more managed policies — LimitExceeded"
  diagnosis: "The principal has reached the 10 managed policy attachment limit."
  resolution: "Consolidate managed policies or convert some to inline policies."

- symptoms: "Inline policy on user is too small for all needed permissions"
  diagnosis: "User inline policies are limited to 2,048 characters."
  resolution: "Use a managed policy (6,144 chars) or attach the inline policy to a role (10,240 chars)."

- symptoms: "Deleted a user but orphaned managed policy remains"
  diagnosis: "Managed policies are standalone and persist after principal deletion."
  resolution: "Delete unused managed policies: `aws iam delete-policy --policy-arn <arn>` (after detaching from all principals)."

## Output Format

```yaml
root_cause: "policy_type — <specific_cause>"
evidence:
  - type: managed_policies
    content: "<list of attached managed policies>"
  - type: inline_policies
    content: "<list of inline policies>"
  - type: limits
    content: "managed: <count>/10, inline_size: <size>/<limit>"
severity: MEDIUM
mitigation:
  immediate: "Consolidate or convert policies as needed"
  long_term: "Standardize on managed policies with ABAC for scalability"
```


## Safety Ratings

safety_ratings:
  - "Policy type migration (converting between managed and inline policies): YELLOW — Detaching one policy type and reattaching as another can cause a brief window where permissions are missing"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the migration involves policies on production principals"
  - "Escalate if the principal is actively serving traffic and cannot tolerate any permission gap"
  - "Escalate if the policy contains complex conditions that must be preserved exactly"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Reattach the original policy type — if a managed policy was detached, reattach it using `aws iam attach-role-policy`; if an inline policy was deleted, recreate it using `aws iam put-role-policy` with the original document
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam list-attached-role-policies` and `aws iam list-role-policies` reveal the full policy attachment model — classification: MEDIUM
- `aws iam get-account-authorization-details` returns all policies and attachments in the account — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- After migrating from inline to managed policy, delete the orphaned inline policy using `aws iam delete-role-policy` or `aws iam delete-user-policy`
- After migrating from managed to inline, consider deleting the managed policy if no longer attached to any principal

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
