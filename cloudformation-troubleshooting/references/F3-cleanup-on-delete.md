---
title: "F3 — Custom Resource Cleanup on Delete"
description: "Diagnose and resolve custom resource failures during stack deletion"
status: active
severity: HIGH
triggers:
  - "custom resource delete"
  - "cleanup failed"
  - "delete custom resource"
  - "stack delete stuck on custom resource"
owner: devops-agent
objective: "Ensure custom resources handle Delete events properly to allow clean stack deletion"
context: "When a stack is deleted, CloudFormation sends a Delete event to custom resources. If the Lambda fails to handle the Delete event or doesn't send a response, the stack deletion hangs for up to 1 hour per custom resource. This is especially problematic because the user is trying to clean up, and the cleanup itself is stuck."
---

## Phase 1 — Triage

MUST:
- Check if the stack is stuck on a custom resource during deletion: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`DELETE_IN_PROGRESS` && (ResourceType==`Custom::*` || ResourceType==`AWS::CloudFormation::CustomResource`)]'`
- Check Lambda logs for the Delete event handling: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Verify the Lambda function still exists (it may have been deleted before the stack)
- Check if the Lambda function handles the Delete RequestType

SHOULD:
- Check if the Lambda function's execution role still has permissions
- Verify network connectivity for VPC Lambda functions
- Check if the resources the Lambda tries to clean up still exist

MAY:
- Check if the Lambda function was modified after stack creation
- Review if the custom resource has dependencies that were already deleted

## Phase 2 — Remediate

MUST:
- If the Lambda function is missing or broken, you may need to wait for the 1-hour timeout
- After timeout, the custom resource will fail, and you can retry deletion with `--retain-resources`
- To prevent future issues: ensure the Lambda handles Delete events gracefully, including when the underlying resource is already gone

SHOULD:
- Fix the Lambda function to handle Delete events:
  - Always send SUCCESS for Delete events, even if the cleanup fails
  - Handle "resource not found" errors gracefully (the resource may already be deleted)
  - Add try/catch that sends SUCCESS on Delete to avoid blocking stack deletion
- Redeploy the Lambda function if it was accidentally deleted

MAY:
- Manually send a SUCCESS response to the pre-signed URL to unblock the deletion (advanced)
- Use the ServiceTimeout property to reduce the wait time for future deployments

## Common Issues

- symptoms: "Stack deletion stuck on custom resource for 1 hour"
  diagnosis: "Lambda function doesn't handle Delete events or doesn't send a response."
  resolution: "Wait for timeout, then retry with --retain-resources. Fix Lambda for future."

- symptoms: "Custom resource Delete fails because the Lambda function was deleted"
  diagnosis: "The Lambda function was deleted before the stack, so CloudFormation can't invoke it."
  resolution: "Recreate the Lambda function temporarily, or wait for timeout and use --retain-resources."

- symptoms: "Custom resource Delete fails trying to clean up a resource that's already gone"
  diagnosis: "The Lambda tries to delete a resource that was already manually deleted."
  resolution: "Add error handling for 'not found' errors in the Delete handler. Always send SUCCESS."

## Output Format

```yaml
root_cause: "cleanup_on_delete — <function_name>: <delete_handler_issue>"
evidence:
  - type: stack_event
    content: "<custom resource delete event>"
  - type: lambda_logs
    content: "<Delete event handling logs>"
severity: HIGH
mitigation:
  immediate: "Wait for timeout and use --retain-resources, or fix Lambda"
  long_term: "Always send SUCCESS for Delete events, handle 'not found' gracefully"
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
