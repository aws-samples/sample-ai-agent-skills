---
title: "C2 — Resource Already Exists"
description: "Diagnose and resolve 'already exists' errors during CloudFormation resource creation"
status: active
severity: MEDIUM
triggers:
  - "already exists"
  - "AlreadyExistsException"
  - "resource already exists"
  - "EntityAlreadyExists"
  - "BucketAlreadyExists"
owner: devops-agent
objective: "Resolve naming conflicts by importing the existing resource, renaming, or removing the conflict"
context: "This error occurs when CloudFormation tries to create a resource with a specific physical name that already exists in the AWS account. Common with resources that have explicit names (S3 buckets, IAM roles, DynamoDB tables). Resources without explicit names get auto-generated unique names and rarely hit this issue."
---

## Phase 1 — Triage

MUST:
- Identify the resource and its physical name from the CREATE_FAILED event: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'`
- Check if the existing resource is managed by another stack: `aws cloudformation describe-stack-resources --physical-resource-id <physical-id>`
- Check if the existing resource was left behind from a previous failed stack deletion

SHOULD:
- Determine if the existing resource can be imported into this stack
- Check if the resource was created manually (not by CloudFormation)
- Review if a previous stack with the same name was deleted but resources were retained

MAY:
- Check if the resource exists in a different region (for global resources like IAM roles, S3 buckets)

## Phase 2 — Remediate

MUST:
- Choose one of these strategies:
  - Remove the explicit name from the template (let CloudFormation auto-generate)
  - Delete the existing resource if it's orphaned
  - Import the existing resource into the stack (requires DeletionPolicy: Retain)
  - Use a different name in the template

SHOULD:
- For import: add the resource to the template with DeletionPolicy: Retain, then use import: `aws cloudformation create-change-set --stack-name <stack> --change-set-name import --change-set-type IMPORT --resources-to-import '[{"ResourceType":"AWS::S3::Bucket","LogicalResourceId":"MyBucket","ResourceIdentifier":{"BucketName":"my-bucket"}}]' --template-body file://template.yaml`
- Avoid hardcoding physical names in templates when possible

MAY:
- Use CloudFormation parameters for resource names to allow flexibility across environments

## Common Issues

- symptoms: "CREATE_FAILED with 'my-bucket already exists in stack other-stack'"
  diagnosis: "Resource is managed by another CloudFormation stack."
  resolution: "Remove from the other stack first (with Retain), then import into this stack."

- symptoms: "CREATE_FAILED with 'Role my-role already exists'"
  diagnosis: "IAM role was created manually or left from a deleted stack."
  resolution: "Delete the orphaned role or import it. Or remove the explicit RoleName from the template."

- symptoms: "CREATE_FAILED with 'Table already exists: my-table'"
  diagnosis: "DynamoDB table exists from a previous deployment."
  resolution: "Import the table or use a different table name."

## Output Format

```yaml
root_cause: "already_exists — <resource_type>: <physical_name>"
evidence:
  - type: stack_event
    content: "<CREATE_FAILED event>"
  - type: existing_resource
    content: "<details of the existing resource and its owner>"
severity: MEDIUM
mitigation:
  immediate: "Import the resource, rename it, or delete the orphan"
  long_term: "Avoid hardcoding physical resource names in templates"
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
