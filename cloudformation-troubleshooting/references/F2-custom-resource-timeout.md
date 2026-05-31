---
title: "F2 — Custom Resource Timeout"
description: "Diagnose and resolve custom resource timeout issues"
status: active
severity: HIGH
triggers:
  - "custom resource timeout"
  - "timed out waiting"
  - "1 hour timeout"
  - "ServiceTimeout"
  - "custom resource stuck"
owner: devops-agent
objective: "Identify why the custom resource timed out and ensure the Lambda sends a response within the timeout period"
context: "Custom resources have a default 1-hour timeout. CloudFormation waits for the Lambda to send a SUCCESS or FAILED response to the pre-signed S3 URL. If no response is received within the timeout, CloudFormation marks the resource as failed. The ServiceTimeout property (1-3600 seconds) can customize this. The Lambda function's own timeout is separate from the custom resource timeout."
---

## Phase 1 — Triage

MUST:
- Confirm the timeout from stack events: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?contains(ResourceStatusReason, `timed out`)]'`
- Check Lambda function timeout setting: `aws lambda get-function-configuration --function-name <function-name> --query '{Timeout:Timeout,MemorySize:MemorySize}'`
- Check Lambda logs for execution: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Determine if the Lambda executed at all, executed but didn't send a response, or timed out itself

SHOULD:
- Check if the Lambda is in a VPC and can reach the S3 pre-signed URL
- Check Lambda invocation metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <time> --end-time <time> --period 300 --statistics Maximum`
- Verify the Lambda function was actually invoked (check Invocations metric)

MAY:
- Check if the Lambda hit concurrency limits
- Check if the pre-signed URL expired before the Lambda could use it

## Phase 2 — Remediate

MUST:
- Ensure the Lambda sends a response in ALL code paths:
  - Success path: send SUCCESS
  - Error/catch path: send FAILED
  - Timeout handler: send FAILED before Lambda times out
- If the Lambda needs more time, increase its timeout (max 15 minutes)
- If the operation needs more than 15 minutes, use Step Functions or async patterns

SHOULD:
- Set ServiceTimeout on the custom resource to a reasonable value instead of the 1-hour default
- Add a timeout handler in the Lambda that sends FAILED before the Lambda itself times out
- Ensure VPC Lambda has network access (NAT gateway or S3 VPC endpoint)

MAY:
- Use async invocation with a callback pattern for long-running operations
- Implement idempotency in the Lambda function for safe retries

## Common Issues

- symptoms: "Stack stuck for 1 hour then custom resource fails"
  diagnosis: "Lambda didn't send a response to the pre-signed URL."
  resolution: "Add cfn-response.send() in all code paths including error handlers."

- symptoms: "Lambda times out at 15 minutes but custom resource waits 1 hour"
  diagnosis: "Lambda timeout and custom resource timeout are independent. Lambda timed out but CloudFormation still waits."
  resolution: "Add a timeout handler that sends FAILED before Lambda times out. Set ServiceTimeout to match."

- symptoms: "Custom resource works locally but times out in VPC"
  diagnosis: "VPC Lambda cannot reach the S3 pre-signed URL."
  resolution: "Add NAT gateway or S3 VPC endpoint."

## Output Format

```yaml
root_cause: "custom_resource_timeout — <function_name>: <lambda_executed: yes|no>, <response_sent: yes|no>"
evidence:
  - type: stack_event
    content: "<timeout event>"
  - type: lambda_logs
    content: "<Lambda execution logs or absence thereof>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda to send response in all code paths"
  long_term: "Set ServiceTimeout, add timeout handlers, ensure VPC connectivity"
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
