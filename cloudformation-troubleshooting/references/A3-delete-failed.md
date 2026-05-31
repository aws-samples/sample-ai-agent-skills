---
title: "A3 — Stack DELETE_FAILED"
description: "Diagnose and resolve CloudFormation stack deletion failures"
status: active
severity: HIGH
triggers:
  - "DELETE_FAILED"
  - "cannot delete stack"
  - "delete failed"
  - "stack stuck deleting"
owner: devops-agent
objective: "Identify which resources cannot be deleted, resolve the dependencies, and complete stack deletion"
context: "DELETE_FAILED means one or more resources could not be deleted. Common causes: S3 bucket not empty, ENI in use, security group referenced by another resource, Lambda@Edge replicas still propagating, or resources with termination protection. The stack remains in DELETE_FAILED and can be retried."
---

## Phase 1 — Triage

MUST:
- Get stack status and identify failing resources: `aws cloudformation describe-stacks --stack-name <stack>`
- Find DELETE_FAILED events: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`DELETE_FAILED`]'`
- List remaining resources: `aws cloudformation list-stack-resources --stack-name <stack> --query 'StackResourceSummaries[?ResourceStatus!=`DELETE_COMPLETE`]'`
- Read the ResourceStatusReason for each DELETE_FAILED resource

SHOULD:
- Check if S3 buckets are non-empty: `aws s3 ls s3://<bucket> --summarize`
- Check if ENIs are in use: `aws ec2 describe-network-interfaces --filters Name=group-id,Values=<sg-id>`
- Check if security groups are referenced: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check for Lambda@Edge replicas: `aws lambda list-functions --query 'Functions[?starts_with(FunctionName, `us-east-1.`)]'`

MAY:
- Check if the resource has termination protection enabled
- Check if the resource is shared with other accounts via RAM

## Phase 2 — Remediate

MUST:
- Resolve the dependency preventing deletion:
  - S3 bucket not empty: `aws s3 rm s3://<bucket> --recursive` then retry delete
  - ENI in use: detach or delete the dependent resource first
  - Security group referenced: remove the reference from the other security group
- Retry the stack deletion: `aws cloudformation delete-stack --stack-name <stack>`
- If resources truly cannot be deleted, skip them: `aws cloudformation delete-stack --stack-name <stack> --retain-resources <logical-id-1> <logical-id-2>`

SHOULD:
- Clean up retained resources manually after the stack is deleted
- Document which resources were retained and why

MAY:
- Add `DeletionPolicy: Delete` with empty-bucket Lambda custom resources for S3 buckets in the template to prevent future issues

## Common Issues

- symptoms: "DELETE_FAILED on S3 bucket with 'The bucket you tried to delete is not empty'"
  diagnosis: "S3 bucket contains objects or versions."
  resolution: "Empty the bucket including all versions: aws s3 rm s3://<bucket> --recursive. If versioned, also delete all versions."

- symptoms: "DELETE_FAILED on security group with 'resource sg-xxx has a dependent object'"
  diagnosis: "Another security group or ENI references this security group."
  resolution: "Find and remove the reference, then retry deletion."

- symptoms: "DELETE_FAILED on Lambda@Edge function"
  diagnosis: "Lambda@Edge replicas take time to delete after disassociating from CloudFront."
  resolution: "Wait up to several hours for replicas to be cleaned up, then retry."

## Output Format

```yaml
root_cause: "delete_failed — <resource_logical_id>: <dependency_cause>"
evidence:
  - type: stack_event
    content: "<DELETE_FAILED event with ResourceStatusReason>"
  - type: resource_dependency
    content: "<dependency details>"
severity: HIGH
mitigation:
  immediate: "Resolve the dependency and retry deletion, or use --retain-resources"
  long_term: "Add DeletionPolicy and cleanup custom resources to templates"
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
