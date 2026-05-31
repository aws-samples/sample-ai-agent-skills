---
title: "F1 — Lambda-Backed Custom Resource Failures"
description: "Diagnose Lambda function failures in CloudFormation custom resources"
status: active
severity: HIGH
triggers:
  - "Custom::*"
  - "AWS::CloudFormation::CustomResource"
  - "custom resource failed"
  - "Lambda custom resource"
  - "cfn-response"
owner: devops-agent
objective: "Identify why the Lambda-backed custom resource failed and fix the function or its configuration"
context: "Lambda-backed custom resources receive a request from CloudFormation (Create/Update/Delete) and must send a SUCCESS or FAILED response to a pre-signed S3 URL. If the Lambda crashes, times out, or fails to send the response, CloudFormation waits up to 1 hour (default) before marking the resource as failed. The most common issue is the Lambda not sending the response."
---

## Phase 1 — Triage

MUST:
- Find the custom resource failure event: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceType==`Custom::*` || ResourceType==`AWS::CloudFormation::CustomResource`]'`
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Verify the Lambda function exists and is configured correctly: `aws lambda get-function --function-name <function-name>`
- Check if the Lambda function timed out (see F2 if timeout)

SHOULD:
- Check the Lambda execution role permissions: `aws iam get-role-policy --role-name <role> --policy-name <policy>`
- Verify the Lambda function can reach the pre-signed S3 URL (VPC configuration)
- Check if the Lambda function uses cfn-response module correctly

MAY:
- Check Lambda dead letter queue for failed invocations
- Review Lambda concurrency limits

## Phase 2 — Remediate

MUST:
- Fix the Lambda function code to handle the error and send a response
- Ensure cfn-response.send() or equivalent is called in ALL code paths (success, error, and catch blocks)
- Ensure the Lambda function has network access to send the response (NAT gateway or VPC endpoint if in VPC)
- Retry the stack operation

SHOULD:
- Add comprehensive error handling with try/catch that always sends a FAILED response
- Add logging for debugging
- Test the Lambda function independently before deploying via CloudFormation

MAY:
- Increase Lambda timeout to accommodate longer operations
- Use the ServiceTimeout property on the custom resource to set a shorter timeout than the default 1 hour

## Common Issues

- symptoms: "Custom resource CREATE_FAILED with no clear error message"
  diagnosis: "Lambda function crashed or didn't send a response. CloudFormation waited 1 hour."
  resolution: "Check Lambda logs. Add error handling that always sends cfn-response."

- symptoms: "Custom resource works on Create but fails on Update or Delete"
  diagnosis: "Lambda function only handles Create events, not Update/Delete."
  resolution: "Handle all RequestType values: Create, Update, Delete."

- symptoms: "Custom resource fails in VPC Lambda"
  diagnosis: "Lambda in VPC cannot reach the pre-signed S3 URL without internet access."
  resolution: "Add NAT gateway or S3 VPC endpoint to the VPC."

## Output Format

```yaml
root_cause: "custom_resource_failure — <function_name>: <error_from_logs>"
evidence:
  - type: stack_event
    content: "<custom resource failure event>"
  - type: lambda_logs
    content: "<relevant Lambda log entries>"
severity: HIGH
mitigation:
  immediate: "Fix the Lambda function and retry"
  long_term: "Add comprehensive error handling and response sending in all code paths"
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
