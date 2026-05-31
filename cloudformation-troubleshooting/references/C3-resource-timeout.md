---
title: "C3 — Resource Timeout"
description: "Diagnose and resolve resource creation or update timeouts in CloudFormation"
status: active
severity: HIGH
triggers:
  - "timed out"
  - "timeout"
  - "WaitCondition"
  - "CreationPolicy timeout"
  - "resource stabilization"
owner: devops-agent
objective: "Identify why a resource timed out during creation or update and fix the underlying issue"
context: "Resource timeouts occur when CloudFormation waits for a resource to reach a stable state but it doesn't within the allowed time. Common with EC2 instances using CreationPolicy/cfn-signal, WaitConditions, ECS services, and custom resources (1-hour default timeout). The timeout period varies by resource type."
---

## Phase 1 — Triage

MUST:
- Find the timeout event: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?contains(ResourceStatusReason, `timed out`) || contains(ResourceStatusReason, `timeout`)]'`
- Identify the resource type and its timeout configuration
- For EC2 with CreationPolicy: check if cfn-signal was sent: `aws cloudformation describe-stack-resource --stack-name <stack> --logical-resource-id <id>`
- For custom resources: check Lambda logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`

SHOULD:
- For EC2 instances: check instance system logs: `aws ec2 get-console-output --instance-id <id>`
- For ECS services: check service events: `aws ecs describe-services --cluster <cluster> --services <service>`
- Check if the resource is in a private subnet without internet access (cfn-signal needs to reach CloudFormation endpoint)

MAY:
- Check VPC endpoint configuration for CloudFormation if in a private subnet
- Review UserData scripts for errors that prevent cfn-signal from running

## Phase 2 — Remediate

MUST:
- Fix the root cause of the timeout:
  - For cfn-signal: ensure the signal is sent after successful initialization
  - For custom resources: ensure the Lambda sends a response to the pre-signed URL
  - For ECS services: fix health check or task definition issues
- Increase the timeout if the resource legitimately needs more time

SHOULD:
- For private subnets: add a VPC endpoint for CloudFormation or NAT gateway
- Add error handling in UserData to send failure signal on errors
- Add CloudWatch alarms for long-running resource creation

MAY:
- Use cfn-init with configsets for structured EC2 initialization
- Add WaitCondition with explicit timeout for complex multi-step provisioning

## Common Issues

- symptoms: "Failed to receive 1 resource signal(s) within the specified duration"
  diagnosis: "EC2 instance did not send cfn-signal within the CreationPolicy timeout."
  resolution: "Check instance UserData, system logs, and network connectivity to CloudFormation endpoint."

- symptoms: "Custom resource timed out after 3600 seconds"
  diagnosis: "Lambda custom resource did not send a response to the pre-signed S3 URL."
  resolution: "Check Lambda logs. Ensure cfn-response.send() is called in all code paths."

- symptoms: "ECS service stuck in CREATE_IN_PROGRESS"
  diagnosis: "ECS tasks failing health checks or crashing repeatedly."
  resolution: "Check ECS task logs and health check configuration."

## Output Format

```yaml
root_cause: "resource_timeout — <logical_id>: <timeout_cause>"
evidence:
  - type: stack_event
    content: "<timeout event>"
  - type: resource_logs
    content: "<instance logs, Lambda logs, or ECS events>"
severity: HIGH
mitigation:
  immediate: "Fix the initialization or signaling issue"
  long_term: "Add proper error handling and monitoring for resource initialization"
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
