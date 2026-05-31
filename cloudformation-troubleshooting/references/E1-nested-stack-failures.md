---
title: "E1 — Nested Stack Failures"
description: "Diagnose failures in nested CloudFormation stacks by tracing to the innermost failure"
status: active
severity: HIGH
triggers:
  - "nested stack"
  - "Embedded stack"
  - "child stack"
  - "AWS::CloudFormation::Stack"
  - "nested stack failed"
owner: devops-agent
objective: "Trace nested stack failures to the innermost root cause and fix the failing resource"
context: "Nested stacks fail from the inside out. The parent stack event only shows 'Embedded stack <arn> was not successfully created/updated.' The actual root cause is in the nested stack's own events. For deeply nested stacks, you must traverse each level to find the innermost failure."
---

## Phase 1 — Triage

MUST:
- Find the nested stack failure in the parent: `aws cloudformation describe-stack-events --stack-name <parent-stack> --query 'StackEvents[?ResourceType==`AWS::CloudFormation::Stack` && ResourceStatus==`CREATE_FAILED`]'`
- Get the nested stack ARN from the PhysicalResourceId of the failed event
- Check the nested stack's events: `aws cloudformation describe-stack-events --stack-name <nested-stack-arn>`
- If the nested stack contains further nested stacks, repeat until you find the actual resource failure
- Read the ResourceStatusReason of the innermost failing resource

SHOULD:
- List all resources in the nested stack: `aws cloudformation list-stack-resources --stack-name <nested-stack-arn>`
- Check if the nested stack template is valid: `aws cloudformation get-template --stack-name <nested-stack-arn>`
- Verify parameters passed from parent to nested stack

MAY:
- Check if the nested stack template URL is accessible
- Verify the nested stack template hasn't been modified in S3

## Phase 2 — Remediate

MUST:
- Fix the root cause in the innermost nested stack
- The fix may be in the nested stack template, the parent stack parameters, or the environment
- Retry the parent stack operation — nested stacks are updated through the parent

SHOULD:
- Test nested stack templates independently before integrating
- Use `aws cloudformation validate-template` on each nested stack template

MAY:
- Consider flattening shallow nested stacks to simplify debugging
- Use stack outputs and cross-stack references instead of deep nesting

## Common Issues

- symptoms: "Parent stack shows 'Embedded stack <arn> was not successfully created'"
  diagnosis: "The nested stack failed. Root cause is in the nested stack events."
  resolution: "Navigate to the nested stack ARN and check its events."

- symptoms: "Nested stack fails with parameter errors"
  diagnosis: "Parent stack passes incorrect parameter values to the nested stack."
  resolution: "Check the parent template's AWS::CloudFormation::Stack Parameters property."

- symptoms: "Nested stack template not found"
  diagnosis: "The TemplateURL in the parent template points to a non-existent or inaccessible S3 object."
  resolution: "Verify the S3 URL and permissions. Upload the template if missing."

## Output Format

```yaml
root_cause: "nested_stack_failure — depth: <n>, innermost_resource: <logical_id>: <reason>"
evidence:
  - type: parent_event
    content: "<parent stack nested stack failure event>"
  - type: nested_event
    content: "<innermost stack root cause event>"
severity: HIGH
mitigation:
  immediate: "Fix the innermost failing resource and retry from the parent"
  long_term: "Test nested stack templates independently and minimize nesting depth"
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
