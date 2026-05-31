---
title: "I1 — StackSet Deployment Failures"
description: "Diagnose failures in Control Tower-managed StackSet deployments"
status: active
severity: HIGH
triggers:
  - "StackSet operation failed"
  - "Stack instance FAILED"
  - "AWSControlTowerBP stack error"
owner: devops-agent
objective: "Identify and resolve StackSet deployment failures"
context: "Control Tower uses CloudFormation StackSets to deploy baseline configurations (Config, CloudTrail, IAM roles) to all governed accounts. StackSet failures mean accounts are missing baseline governance. CT StackSets are prefixed with AWSControlTowerBP-."
---

## Phase 1 — Triage

MUST:
- List CT StackSets: `aws cloudformation list-stack-sets --status ACTIVE` (filter for AWSControlTowerBP-*)
- Check for failed stack instances: `aws cloudformation list-stack-instances --stack-set-name <name> --filters Name=DETAILED_STATUS,Values=FAILED`
- Get failure details: `aws cloudformation describe-stack-instance --stack-set-name <name> --stack-instance-account <account> --stack-instance-region <region>`
- Check the specific CloudFormation error in the target account

SHOULD:
- Identify which accounts and regions are affected
- Check if the failure is consistent across accounts or isolated
- Verify the AWSControlTowerExecution role exists in failed accounts

MAY:
- Check StackSet operation history: `aws cloudformation list-stack-set-operations --stack-set-name <name>`
- Review the StackSet template for issues

## Phase 2 — Remediate

MUST:
- Fix the underlying issue in the target account (role, resource conflict, SCP)
- Retry the StackSet operation or re-register the OU
- Do NOT modify CT StackSet templates directly

SHOULD:
- Delete failed stack instances before retrying
- Verify the fix resolves the issue in all affected accounts
- Check other CT StackSets for similar failures

MAY:
- Increase StackSet operation concurrency for faster deployment
- Contact AWS Support for persistent StackSet failures

## Common Issues

- symptoms: "Stack instance FAILED with access denied"
  diagnosis: "AWSControlTowerExecution role missing or SCP blocking in the target account."
  resolution: "Create the role or fix the SCP, then re-register the OU."

- symptoms: "Stack instance FAILED with resource already exists"
  diagnosis: "A resource CT is trying to create already exists (Config recorder, IAM role)."
  resolution: "Delete the conflicting resource in the target account, then retry."

- symptoms: "StackSet operation times out"
  diagnosis: "Too many accounts or slow resource creation."
  resolution: "Check individual stack instance status. Increase operation timeout if possible."

## Safety Ratings

```
safety_ratings:
  - "list-stack-sets: GREEN — read-only StackSet listing"
  - "list-stack-instances: GREEN — read-only instance listing"
  - "describe-stack-instance: GREEN — read-only instance details"
  - "delete failed stack instances: YELLOW — recoverable, CT will recreate on re-register"
  - "re-register OU to redeploy StackSets: YELLOW — recoverable, resets CT baselines"
  - "modify CT StackSet templates directly: RED — causes drift, breaks CT governance"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- StackSet failures affect baseline governance (Config, CloudTrail, IAM roles)
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- Persistent StackSet failures with no clear root cause

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: StackSet instance details (account IDs, regions, failure reasons)
- MEDIUM: StackSet template content (expose baseline configurations)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If failed stack instances were deleted, document which accounts/regions were affected before re-registering
- Do NOT modify CT StackSet templates directly — always use CT mechanisms (re-register OU, repair landing zone)
- If a resource conflict caused the failure, document the conflicting resource before deleting it

## Output Format

```yaml
root_cause: "stackset_failure — <stackset_name> in <account>/<region>"
evidence:
  - type: stack_instance
    content: "<failure status and error>"
severity: HIGH
mitigation:
  immediate: "Fix the target account issue and re-register OU"
  long_term: "Monitor StackSet health and pre-validate accounts"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
