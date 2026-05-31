---
title: "C1 — Resource Creation Failure"
description: "Diagnose individual resource creation failures within a CloudFormation stack"
status: active
severity: HIGH
triggers:
  - "CREATE_FAILED"
  - "resource failed to create"
  - "Resource handler returned message"
  - "resource creation error"
owner: devops-agent
objective: "Identify why a specific resource failed to create and fix the template or environment"
context: "Individual resource creation failures are the most common cause of stack failures. The ResourceStatusReason in the stack event contains the specific error from the underlying AWS service. Common causes: insufficient permissions, invalid property values, service quotas, missing prerequisites, and naming conflicts."
---

## Phase 1 — Triage

MUST:
- Find the CREATE_FAILED event for the specific resource: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED` && LogicalResourceId==`<logical-id>`]'`
- Read the ResourceStatusReason — this contains the actual AWS service error
- Check the resource properties in the template: `aws cloudformation get-template --stack-name <stack>`
- Verify IAM permissions for the resource type

SHOULD:
- Check service quotas: `aws service-quotas get-service-quota --service-code <service> --quota-code <code>`
- Check CloudTrail for the failed API call: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Create<ResourceType>`
- Verify prerequisites exist (VPC, subnet, security group, KMS key, etc.)

MAY:
- Check if the resource was partially created and needs manual cleanup
- Review AWS Health Dashboard for service issues in the region

## Phase 2 — Remediate

MUST:
- Fix the root cause based on the ResourceStatusReason
- For permission errors: add the required IAM action to the execution role
- For invalid properties: correct the property value in the template
- For quota limits: request a quota increase or reduce resource count
- For missing prerequisites: create the prerequisite resource or fix the reference

SHOULD:
- Use `--disable-rollback` to preserve the failed state for debugging
- Test the fix with a change set before applying

MAY:
- Add condition checks in the template to validate prerequisites
- Use custom resources for pre-flight validation

## Common Issues

- symptoms: "CREATE_FAILED with 'You are not authorized to perform this operation'"
  diagnosis: "CloudFormation execution role lacks the required permission."
  resolution: "Add the specific IAM action to the execution role."

- symptoms: "CREATE_FAILED with 'Limit exceeded' or 'quota exceeded'"
  diagnosis: "AWS service quota reached."
  resolution: "Request a quota increase via Service Quotas console or reduce resource count."

- symptoms: "CREATE_FAILED with 'Invalid parameter value' or 'Malformed'"
  diagnosis: "A resource property has an invalid value."
  resolution: "Check the property value against the resource type documentation."

## Output Format

```yaml
root_cause: "resource_creation_failure — <logical_id> (<resource_type>): <status_reason>"
evidence:
  - type: stack_event
    content: "<CREATE_FAILED event with ResourceStatusReason>"
  - type: cloudtrail
    content: "<failed API call details>"
severity: HIGH
mitigation:
  immediate: "Fix the resource configuration and retry"
  long_term: "Add IAM policy validation and quota checks to deployment pipeline"
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
