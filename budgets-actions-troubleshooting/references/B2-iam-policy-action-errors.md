---
title: "B2 — IAM Policy Action Errors"
description: "Diagnose IAM policy attachment budget action errors"
status: active
severity: HIGH
triggers: ["iam policy action", "policy attachment failed", "deny policy action", "iam budget action"]
owner: devops-agent
objective: "Resolve IAM policy attachment action errors"
context: "IAM policy actions attach a specified IAM policy to users, groups, or roles when budget thresholds are breached. The policy must pre-exist."
---

## Phase 1 — Triage

MUST:
- Check action configuration: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify the IAM policy exists: `aws iam get-policy --policy-arn <policy-arn>`
- Check execution role permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names iam:AttachUserPolicy iam:AttachGroupPolicy iam:AttachRolePolicy`

SHOULD:
- Verify target users/groups/roles exist
- Check if the policy is already attached to targets
- Review the policy content for correctness: `aws iam get-policy-version --policy-arn <arn> --version-id <version>`

MAY:
- Test policy attachment manually in a non-production environment
- Check for conflicting policies on the targets

## Phase 2 — Remediate

MUST:
- Create the IAM deny policy if it does not exist
- Grant iam:AttachUserPolicy, iam:AttachGroupPolicy, or iam:AttachRolePolicy to the execution role
- Verify target IAM entities exist and are accessible

SHOULD:
- Test the deny policy impact before enabling the action
- Set up approval workflow for IAM policy actions

MAY:
- Use permission boundaries instead of deny policies for more granular control
- Implement automated policy detachment after budget period

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget-action: GREEN — read-only action check"
  - "iam get-policy: GREEN — read-only policy check"
  - "iam simulate-principal-policy: GREEN — read-only permission test"
  - "iam attach-user-policy: RED — modifies user permissions, can block access"
```

## Escalation Conditions

- IAM policy action blocks critical service accounts
- Policy attachment affects production IAM roles
- Deny policy is too broad and blocks essential operations
- Policy detachment requires elevated permissions not available

## Data Sensitivity

- IAM policy content: HIGH — reveals permission restrictions
- Target users/groups/roles: MEDIUM — reveals identity structure
- Action configuration: MEDIUM — reveals cost control strategy
- Execution role: MEDIUM — reveals permission scope

## Prohibited Actions

- NEVER create deny-all policies without understanding the impact
- NEVER attach policies to service-linked roles
- NEVER test IAM policy actions on production IAM entities
- NEVER leave deny policies attached after the budget period without review

## Phase 3 — Rollback

1. Detach the IAM policy from affected entities: `aws iam detach-user-policy --user-name <name> --policy-arn <arn>`
2. Detach from groups: `aws iam detach-group-policy --group-name <name> --policy-arn <arn>`
3. Detach from roles: `aws iam detach-role-policy --role-name <name> --policy-arn <arn>`
4. Reset the budget action: `aws budgets execute-budget-action --account-id <id> --budget-name <name> --action-id <id> --execution-type RESET_BUDGET_ACTION`
5. Verify affected entities have correct permissions after detachment

## Common Issues

- symptoms: "IAM policy action fails with NoSuchEntity"
  diagnosis: "Target IAM user, group, or role does not exist or policy ARN is incorrect."
  resolution: "Verify target IAM entities exist. Check policy ARN is correct and the policy has not been deleted."

## Output Format

```yaml
root_cause: "iam_policy_action — <specific_cause>"
evidence:
  - type: policy_action_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM policy action configuration"
  long_term: "Implement approval workflows for IAM policy actions"
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
