---
title: "G1 — StackSet Operation Failures"
description: "Diagnose and resolve CloudFormation StackSet operation failures"
status: active
severity: HIGH
triggers:
  - "StackSet operation failed"
  - "stack set failed"
  - "FAILED operation"
  - "StackSet STOPPED"
owner: devops-agent
objective: "Identify why a StackSet operation failed and fix the template, permissions, or target configuration"
context: "StackSet operations (create, update, delete) deploy to multiple accounts and regions. Operations can fail due to: insufficient permissions in target accounts, template errors, service quotas, or exceeding failure tolerance. Operations are asynchronous and results must be checked per account/region."
---

## Phase 1 — Triage

MUST:
- Get the operation status: `aws cloudformation describe-stack-set-operation --stack-set-name <name> --operation-id <id>`
- Get per-account/region results: `aws cloudformation list-stack-set-operation-results --stack-set-name <name> --operation-id <id>`
- Identify which accounts/regions failed and the failure reasons
- Check if the operation was STOPPED due to exceeding failure tolerance

SHOULD:
- Check the StackSet administration role: `aws iam get-role --role-name AWSCloudFormationStackSetAdministrationRole`
- Check the execution role in the target account: `aws iam get-role --role-name AWSCloudFormationStackSetExecutionRole`
- Verify the StackSet template: `aws cloudformation describe-stack-set --stack-set-name <name> --query 'StackSet.TemplateBody'`

MAY:
- Check if the target accounts are in the correct organizational units
- Review StackSet operation preferences (concurrency, failure tolerance)

## Phase 2 — Remediate

MUST:
- Fix the root cause for the failing accounts/regions
- For permission errors: ensure the execution role exists in target accounts with correct trust policy
- For template errors: fix the template and retry the operation
- For quota errors: request quota increases in the affected accounts/regions
- Retry the operation: `aws cloudformation create-stack-instances --stack-set-name <name> --accounts <account-ids> --regions <regions>`

SHOULD:
- Increase failure tolerance if some failures are acceptable: `--operation-preferences FailureToleranceCount=<n>`
- Reduce concurrency if hitting API throttling: `--operation-preferences MaxConcurrentCount=<n>`

MAY:
- Use service-managed StackSets with AWS Organizations for automatic account enrollment
- Deploy to a subset of accounts first as a canary

## Common Issues

- symptoms: "FAILED with 'AWSCloudFormationStackSetExecutionRole does not exist'"
  diagnosis: "The execution role is missing in the target account."
  resolution: "Create the execution role in the target account with the correct trust policy."

- symptoms: "Operation STOPPED after N failures"
  diagnosis: "Failure tolerance exceeded. Too many accounts/regions failed."
  resolution: "Fix the failing accounts, then retry. Or increase FailureToleranceCount."

- symptoms: "FAILED with 'Account <id> is not authorized'"
  diagnosis: "The administration role cannot assume the execution role in the target account."
  resolution: "Update the execution role trust policy to allow the administration role."

## Output Format

```yaml
root_cause: "stackset_operation_failure — <operation_id>: <failure_count>/<total> failed"
evidence:
  - type: operation_results
    content: "<per-account/region failure reasons>"
  - type: role_config
    content: "<administration and execution role details>"
severity: HIGH
mitigation:
  immediate: "Fix failing accounts and retry the operation"
  long_term: "Use service-managed StackSets and validate roles before deployment"
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
