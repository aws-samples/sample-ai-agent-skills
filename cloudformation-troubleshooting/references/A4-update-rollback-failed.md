---
title: "A4 — Stack UPDATE_ROLLBACK_FAILED"
description: "Recover a stack stuck in UPDATE_ROLLBACK_FAILED state"
status: active
severity: CRITICAL
triggers:
  - "UPDATE_ROLLBACK_FAILED"
  - "rollback failed"
  - "stuck in rollback"
  - "continue update rollback"
owner: devops-agent
objective: "Recover the stack from UPDATE_ROLLBACK_FAILED by identifying resources that cannot be rolled back and using ContinueUpdateRollback with resources to skip"
context: "UPDATE_ROLLBACK_FAILED is the worst stack state. It means an update failed AND the rollback also failed. The stack is stuck and cannot be updated or deleted normally. The only recovery is ContinueUpdateRollback with --resources-to-skip, or deleting the entire stack. There is NO automatic recovery."
---

## Phase 1 — Triage

MUST:
- Confirm the stack is in UPDATE_ROLLBACK_FAILED: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].StackStatus'`
- Find the resources that failed during rollback: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`UPDATE_FAILED`]'`
- List all resources and their current status: `aws cloudformation list-stack-resources --stack-name <stack>`
- Identify which resources need to be skipped (those that cannot return to their previous state)

SHOULD:
- Check if the original update changed resource physical IDs (replacements)
- Check if resources were manually modified outside CloudFormation during the update
- Review CloudTrail for manual changes to stack resources

MAY:
- Check if nested stacks are also in a failed state
- Verify the execution role still has the required permissions

## Phase 2 — Remediate

MUST:
- Attempt ContinueUpdateRollback with resources to skip: `aws cloudformation continue-update-rollback --stack-name <stack> --resources-to-skip <logical-id-1> <logical-id-2>`
- If using an execution role: `aws cloudformation continue-update-rollback --stack-name <stack> --resources-to-skip <logical-id-1> --role-arn <role-arn>`
- Verify the stack transitions to UPDATE_ROLLBACK_COMPLETE: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].StackStatus'`
- After recovery, the skipped resources may be out of sync — verify their actual state

SHOULD:
- After recovery, run drift detection to identify out-of-sync resources: `aws cloudformation detect-stack-drift --stack-name <stack>`
- Update the template to match the actual state of skipped resources
- Perform a corrective update to bring the stack back in sync

MAY:
- If ContinueUpdateRollback fails, the last resort is deleting the stack
- Before deleting, export any important resource identifiers for recreation

## Common Issues

- symptoms: "ContinueUpdateRollback fails with 'Resource <id> does not support UPDATE_ROLLBACK'"
  diagnosis: "The resource type doesn't support rollback for the change that was made."
  resolution: "Add the resource to --resources-to-skip and retry."

- symptoms: "Stack stuck after ContinueUpdateRollback"
  diagnosis: "Additional resources also need to be skipped."
  resolution: "Check events for new failures and add those resources to --resources-to-skip."

- symptoms: "UPDATE_ROLLBACK_FAILED after a nested stack update"
  diagnosis: "The nested stack itself may be in a failed state."
  resolution: "Fix the nested stack first, then retry ContinueUpdateRollback on the parent."

## Output Format

```yaml
root_cause: "update_rollback_failed — resources_to_skip: [<logical-ids>]"
evidence:
  - type: stack_event
    content: "<UPDATE_FAILED events during rollback>"
  - type: resource_status
    content: "<current status of problematic resources>"
severity: CRITICAL
mitigation:
  immediate: "ContinueUpdateRollback with --resources-to-skip"
  long_term: "Run drift detection and corrective update after recovery"
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
