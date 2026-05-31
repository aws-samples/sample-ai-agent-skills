---
title: "H1 — Change Set Failures"
description: "Diagnose and resolve CloudFormation change set creation or execution failures"
status: active
severity: MEDIUM
triggers:
  - "change set failed"
  - "FAILED change set"
  - "change set creation failed"
  - "change set execution failed"
owner: devops-agent
objective: "Identify why a change set failed to create or execute and fix the template or parameters"
context: "Change sets can fail at two stages: creation (template validation, parameter issues) or execution (runtime resource failures). A change set with status FAILED during creation means the template or parameters are invalid. A change set that fails during execution means the planned changes couldn't be applied."
---

## Phase 1 — Triage

MUST:
- Check the change set status: `aws cloudformation describe-change-set --stack-name <stack> --change-set-name <name>`
- If creation failed: read the StatusReason for the validation error
- If execution failed: check stack events for the resource failure: `aws cloudformation describe-stack-events --stack-name <stack>`
- Check if the change set has no changes (status FAILED with "didn't contain changes"): this is informational, not an error

SHOULD:
- Validate the template independently: `aws cloudformation validate-template --template-body file://template.yaml`
- Compare the change set template with the current stack template
- Check if parameters changed between creation and execution

MAY:
- List all change sets for the stack: `aws cloudformation list-change-sets --stack-name <stack>`
- Check if multiple change sets conflict

## Phase 2 — Remediate

MUST:
- For creation failures: fix the template or parameter errors and create a new change set
- For execution failures: treat as a stack update failure (see A2)
- Delete failed change sets: `aws cloudformation delete-change-set --stack-name <stack> --change-set-name <name>`

SHOULD:
- Always review change set details before executing
- Pay attention to Replacement: True entries — these will recreate resources

MAY:
- Use change sets in CI/CD pipelines for automated review gates
- Create change sets with `--include-nested-stacks` for nested stack visibility

## Common Issues

- symptoms: "Change set status FAILED with 'The submitted information didn't contain changes'"
  diagnosis: "The template and parameters are identical to the current stack."
  resolution: "This is not an error. No update is needed."

- symptoms: "Change set creation FAILED with parameter errors"
  diagnosis: "Parameter values don't match constraints or required parameters are missing."
  resolution: "Fix parameter values and create a new change set."

- symptoms: "Change set execution failed but creation succeeded"
  diagnosis: "Runtime conditions prevented the changes (permissions, quotas, etc.)."
  resolution: "Check stack events for the specific resource failure. Fix and retry."

## Output Format

```yaml
root_cause: "change_set_failure — stage: <creation|execution>: <reason>"
evidence:
  - type: change_set_status
    content: "<change set details and StatusReason>"
  - type: stack_events
    content: "<execution failure events if applicable>"
severity: MEDIUM
mitigation:
  immediate: "Fix the template/parameters and create a new change set"
  long_term: "Integrate change set review into CI/CD pipeline"
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
