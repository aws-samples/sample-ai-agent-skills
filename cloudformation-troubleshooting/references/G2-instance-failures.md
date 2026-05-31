---
title: "G2 — StackSet Instance Failures"
description: "Diagnose and resolve individual StackSet stack instance failures"
status: active
severity: HIGH
triggers:
  - "stack instance failed"
  - "OUTDATED instance"
  - "INOPERABLE instance"
  - "instance status"
owner: devops-agent
objective: "Identify why specific stack instances failed and bring them back to a healthy state"
context: "A StackSet instance is a stack in a specific account/region. Instances can be in states: CURRENT (in sync), OUTDATED (needs update), INOPERABLE (failed and needs manual intervention). Instance failures are independent — one account/region can fail while others succeed."
---

## Phase 1 — Triage

MUST:
- List stack instances and their status: `aws cloudformation list-stack-instances --stack-set-name <name> --query 'Summaries[?Status!=`CURRENT`]'`
- Get details for the failing instance: `aws cloudformation describe-stack-instance --stack-set-name <name> --stack-instance-account <account> --stack-instance-region <region>`
- Check the StatusReason for the specific failure
- Check the actual stack in the target account: `aws cloudformation describe-stacks --stack-name <stack-id> --region <region>` (using target account credentials)

SHOULD:
- Check the stack events in the target account for the root cause
- Verify the execution role permissions in the target account
- Check if the target account has the required service quotas

MAY:
- Compare the failing instance with a successful instance in another account/region
- Check if the failure is region-specific (service availability)

## Phase 2 — Remediate

MUST:
- Fix the root cause in the target account/region
- For OUTDATED instances: retry the update: `aws cloudformation update-stack-instances --stack-set-name <name> --accounts <account> --regions <region>`
- For INOPERABLE instances: delete and recreate: `aws cloudformation delete-stack-instances --stack-set-name <name> --accounts <account> --regions <region> --no-retain-stacks` then recreate

SHOULD:
- Fix the underlying issue before retrying (permissions, quotas, etc.)
- Test the fix in one account before rolling out to all

MAY:
- Use StackSet operation preferences to control retry behavior
- Implement account-level pre-checks before StackSet deployment

## Common Issues

- symptoms: "Instance OUTDATED with 'Resource limit exceeded'"
  diagnosis: "Target account hit a service quota in that region."
  resolution: "Request quota increase in the target account/region, then retry."

- symptoms: "Instance INOPERABLE after failed update"
  diagnosis: "The stack in the target account is in a failed state."
  resolution: "Fix the stack in the target account directly, or delete and recreate the instance."

- symptoms: "Instance OUTDATED but retry keeps failing"
  diagnosis: "Persistent issue in the target account (permissions, network, etc.)."
  resolution: "Check the stack events in the target account for the specific error."

## Output Format

```yaml
root_cause: "instance_failure — account: <id>, region: <region>: <status_reason>"
evidence:
  - type: instance_status
    content: "<stack instance details>"
  - type: stack_events
    content: "<target account stack events>"
severity: HIGH
mitigation:
  immediate: "Fix the target account issue and retry or recreate the instance"
  long_term: "Implement pre-deployment validation for target accounts"
```


## Safety Ratings
- GREEN: describe-stacks, describe-stack-events, get-template, validate-template
- YELLOW: continue-update-rollback, create-change-set, update-stack
- RED: delete-stack, set-stack-policy with open policy, skip resources during rollback

## Escalation Conditions
- "Stack is in UPDATE_ROLLBACK_FAILED state"
- "Remediation requires skipping resources during rollback"
- "Fix involves modifying stack policy"
- "Nested stack failure in production"

## Rollback
- Pre-change: "Create change set before any update"
- Verification: "Check stack events after operation"
- Revert: "Use rollback triggers or manual rollback"

## Data Sensitivity
- MEDIUM: "Stack templates may contain hardcoded credentials or account IDs"
- MEDIUM: "Stack outputs may expose endpoint URLs and resource IDs"

## Prohibited Actions
- NEVER leave a stack policy open after temporary modification
- NEVER suggest deleting a production stack to fix UPDATE_ROLLBACK_FAILED without exploring ContinueUpdateRollback
- NEVER suggest `--resources-to-skip` without documenting which resources and why
- NEVER suggest modifying CT-managed stacks (AWSControlTower* prefix)

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
  - "NEVER leave stack policy open after temporary modification"
  - "NEVER delete a stack without exploring ContinueUpdateRollback first"
  - "NEVER suggest disabling rollback on production stacks"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
