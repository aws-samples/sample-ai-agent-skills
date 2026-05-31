---
title: "A1 — Stack CREATE_FAILED"
description: "Diagnose and resolve CloudFormation stack creation failures"
status: active
severity: HIGH
triggers:
  - "CREATE_FAILED"
  - "ROLLBACK_COMPLETE"
  - "stack creation failed"
  - "failed to create stack"
owner: devops-agent
objective: "Identify the root cause resource failure and fix the template or environment to allow successful stack creation"
context: "CREATE_FAILED means one or more resources failed to create. CloudFormation automatically rolls back all created resources. The stack ends in ROLLBACK_COMPLETE. The root cause is the FIRST *_FAILED event in the stack events list (read bottom-up)."
---

## Phase 1 — Triage

MUST:
- Get stack status and failure reason: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].{Status:StackStatus,Reason:StackStatusReason}'`
- Get all stack events and find the FIRST failure (read bottom-up): `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'`
- Identify the failing resource logical ID and status reason from the first CREATE_FAILED event
- Validate the template for syntax issues: `aws cloudformation validate-template --template-body file://template.yaml`

SHOULD:
- Check IAM permissions for the CloudFormation execution role: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names <failing-action>`
- Check service quotas for the failing resource type: `aws service-quotas get-service-quota --service-code <service> --quota-code <code>`
- Review CloudTrail for the specific API call that failed: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=<service>.amazonaws.com --max-results 10`

MAY:
- Check if the stack was created with `--disable-rollback` to preserve failed resources for debugging
- Review parameter values for correctness: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].Parameters'`

## Phase 2 — Remediate

MUST:
- Fix the root cause identified in the first CREATE_FAILED event
- For IAM issues: add required permissions to the CloudFormation execution role
- For resource-specific errors: fix the template property or environment prerequisite
- Delete the ROLLBACK_COMPLETE stack before retrying: `aws cloudformation delete-stack --stack-name <stack>`
- Retry creation with the fixed template

SHOULD:
- Use `--disable-rollback` during debugging to preserve failed state for inspection
- Add `--on-failure DO_NOTHING` to keep resources for investigation

MAY:
- Enable CloudFormation termination protection after successful creation: `aws cloudformation update-termination-protection --enable-termination-protection --stack-name <stack>`

## Common Issues

- symptoms: "CREATE_FAILED with 'API: ec2:RunInstances You are not authorized'"
  diagnosis: "CloudFormation execution role lacks EC2 permissions."
  resolution: "Add ec2:RunInstances permission to the execution role."

- symptoms: "CREATE_FAILED with 'Resource limit exceeded'"
  diagnosis: "AWS service quota reached for the resource type."
  resolution: "Request a quota increase or reduce resource count."

- symptoms: "CREATE_FAILED with 'Template format error' before any resources"
  diagnosis: "Template has syntax or structural errors."
  resolution: "Run validate-template and fix the reported errors."

## Output Format

```yaml
root_cause: "create_failed — <resource_logical_id>: <status_reason>"
evidence:
  - type: stack_event
    content: "<first CREATE_FAILED event with ResourceStatusReason>"
  - type: template_validation
    content: "<validate-template output if relevant>"
severity: HIGH
mitigation:
  immediate: "Fix the failing resource configuration and retry stack creation"
  long_term: "Add pre-creation validation and use change sets for complex stacks"
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
