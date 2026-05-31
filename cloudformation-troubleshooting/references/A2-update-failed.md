---
title: "A2 — Stack UPDATE_FAILED"
description: "Diagnose and resolve CloudFormation stack update failures"
status: active
severity: HIGH
triggers:
  - "UPDATE_FAILED"
  - "UPDATE_ROLLBACK_COMPLETE"
  - "stack update failed"
  - "update rollback"
owner: devops-agent
objective: "Identify why the stack update failed, ensure rollback completed, and fix the template or environment for a successful update"
context: "UPDATE_FAILED means a resource change failed during a stack update. CloudFormation automatically rolls back to the previous state. The stack ends in UPDATE_ROLLBACK_COMPLETE. If rollback also fails, the stack enters UPDATE_ROLLBACK_FAILED (see A4)."
---

## Phase 1 — Triage

MUST:
- Get stack status: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].{Status:StackStatus,Reason:StackStatusReason}'`
- Find the first UPDATE_FAILED event (read bottom-up): `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`UPDATE_FAILED`]'`
- Check if the update triggered a resource replacement: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`UPDATE_IN_PROGRESS`].{LogicalId:LogicalResourceId,Reason:ResourceStatusReason}'`
- Verify the stack is in UPDATE_ROLLBACK_COMPLETE (rollback succeeded)

SHOULD:
- Compare the current template with the attempted update: `aws cloudformation get-template --stack-name <stack> --template-stage Original`
- Check if a stack policy blocked the update: `aws cloudformation get-stack-policy --stack-name <stack>`
- Review the change set that was executed: `aws cloudformation describe-change-set --stack-name <stack> --change-set-name <changeset>`

MAY:
- Check CloudTrail for the specific API call that failed during the update
- Review resource-level events for resources that were being replaced

## Phase 2 — Remediate

MUST:
- Fix the root cause from the first UPDATE_FAILED event
- For replacement failures: ensure the new resource can be created before the old one is deleted
- For permission errors: update the execution role with required permissions
- Create a change set to preview the fix before applying: `aws cloudformation create-change-set --stack-name <stack> --template-body file://template.yaml --change-set-name fix-update`

SHOULD:
- Review the change set for unexpected replacements before executing
- Test the update in a non-production environment first

MAY:
- Temporarily modify the stack policy to allow the update: `aws cloudformation set-stack-policy --stack-name <stack> --stack-policy-body '{"Statement":[{"Effect":"Allow","Action":"Update:*","Principal":"*","Resource":"*"}]}'`
- IMPORTANT: If you modify the stack policy, you MUST restore the original policy immediately after the update completes or fails:
  1. Before modifying: save the current policy: `aws cloudformation get-stack-policy --stack-name <stack> > original-policy.json`
  2. After update: restore it: `aws cloudformation set-stack-policy --stack-name <stack> --stack-policy-body file://original-policy.json`
- Better alternative: use `--stack-policy-during-update-body` which only applies during the update and auto-reverts: `aws cloudformation update-stack --stack-name <stack> --template-body file://template.yaml --stack-policy-during-update-body '{"Statement":[{"Effect":"Allow","Action":"Update:Modify","Principal":"*","Resource":"LogicalResourceId/<resource>"}]}'`

## Common Issues

- symptoms: "UPDATE_FAILED with 'Resource handler returned message: ... already exists'"
  diagnosis: "Update tried to create a replacement resource but the physical resource already exists."
  resolution: "Delete the conflicting resource or use a different name/identifier."

- symptoms: "UPDATE_FAILED with 'Action denied by stack policy'"
  diagnosis: "Stack policy prevents updates to the resource."
  resolution: "Modify the stack policy to allow the update, or use --stack-policy-during-update-body."

- symptoms: "UPDATE_FAILED on a resource replacement with data loss"
  diagnosis: "Resource replacement deletes the old resource. Stateful resources (RDS, DynamoDB) lose data."
  resolution: "Use UpdateReplacePolicy: Snapshot or Retain. Create a backup before updating."

## Output Format

```yaml
root_cause: "update_failed — <resource_logical_id>: <status_reason>"
evidence:
  - type: stack_event
    content: "<first UPDATE_FAILED event>"
  - type: change_set
    content: "<change set details showing the planned change>"
severity: HIGH
mitigation:
  immediate: "Fix the failing resource and retry with a change set"
  long_term: "Always preview updates with change sets before executing"
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
