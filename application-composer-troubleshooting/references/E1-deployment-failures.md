---
title: "E1 — Deployment Failures"
description: "Diagnose deployment failures for Application Composer projects"
status: active
severity: CRITICAL
triggers: ["deployment failed", "sam deploy error", "stack creation failed", "stack update failed"]
owner: devops-agent
objective: "Resolve deployment failures for Application Composer-generated templates"
context: "Deployments go through CloudFormation (via SAM CLI). Deployment failures are CloudFormation failures."
---

## Phase 1 — Triage

MUST:
- Check stack status: `aws cloudformation describe-stacks --stack-name <name> --query 'Stacks[0].StackStatus'`
- Review failed events: `aws cloudformation describe-stack-events --stack-name <name> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED` || ResourceStatus==`UPDATE_FAILED`].[LogicalResourceId,ResourceStatusReason]'`
- Verify IAM permissions for deployment: `aws sts get-caller-identity`

SHOULD:
- Check CloudFormation service limits: `aws cloudformation describe-account-limits`
- Review the deployment parameters: `aws cloudformation describe-stacks --stack-name <name> --query 'Stacks[0].Parameters'`

MAY:
- Check CloudTrail for deployment-related events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudformation.amazonaws.com --max-results 20`
- Review S3 deployment bucket: `aws s3 ls s3://<deployment-bucket>/`

## Phase 2 — Remediate

MUST:
- Fix the specific resource that caused the failure (from stack events)
- Ensure IAM role has permissions to create all resource types in the template
- Resolve parameter value errors

SHOULD:
- Use change sets for production updates: `aws cloudformation create-change-set --stack-name <name> --template-body file://template.yaml --change-set-name fix`
- Deploy to a staging environment first

MAY:
- Enable CloudFormation termination protection for production stacks
- Set up CloudFormation stack notifications via SNS

## Safety Ratings

```
safety_ratings:
  - "cloudformation describe-stacks: GREEN — read-only stack status"
  - "cloudformation describe-stack-events: GREEN — read-only event history"
  - "cloudformation create-change-set: YELLOW — creates change set preview"
  - "sam deploy: RED — modifies AWS infrastructure"
```

## Escalation Conditions

- Stack is stuck in UPDATE_ROLLBACK_FAILED or DELETE_FAILED state
- Deployment failure causes data loss in stateful resources
- IAM permission issues require organization-level policy changes
- Deployment affects production traffic or availability

## Data Sensitivity

- `aws cloudformation describe-stacks`: MEDIUM — reveals stack parameters and outputs
- `aws cloudformation describe-stack-events`: MEDIUM — reveals resource creation details
- `aws sts get-caller-identity`: LOW — reveals account and principal information
- Deployment parameters: HIGH — may contain database passwords or API keys

## Prohibited Actions

- NEVER delete a production stack to fix deployment errors
- NEVER use --disable-rollback in production
- NEVER deploy without reviewing the change set in production
- NEVER hardcode secrets in template parameters — use SSM Parameter Store or Secrets Manager

## Phase 3 — Rollback

1. If stack update failed, verify automatic rollback completed: `aws cloudformation describe-stacks --stack-name <name>`
2. For UPDATE_ROLLBACK_FAILED: `aws cloudformation continue-update-rollback --stack-name <name>`
3. If rollback fails, skip problematic resources: `aws cloudformation continue-update-rollback --stack-name <name> --resources-to-skip <resource-id>`
4. Restore the previous template version and redeploy
5. Verify all resources are in expected state after rollback

## Common Issues

- symptoms: "sam deploy fails with 'Error: Failed to create changeset'"
  diagnosis: "Template has errors or the stack is in a non-updatable state."
  resolution: "Check stack status. If ROLLBACK_COMPLETE, delete and recreate. If UPDATE_ROLLBACK_FAILED, continue rollback first."

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: stack_event
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix deployment error"
  long_term: "Implement staging environment and change set reviews"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
