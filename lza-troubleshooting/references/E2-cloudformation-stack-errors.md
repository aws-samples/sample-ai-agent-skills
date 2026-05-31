---
title: "E2 — CloudFormation Stack Errors"
description: "Diagnose and resolve CloudFormation stack deployment failures in LZA"
status: active
severity: HIGH
triggers: ["CloudFormation stack failed", "stack creation error", "stack update failed", "ROLLBACK_COMPLETE", "CREATE_FAILED"]
owner: devops-agent
objective: "Resolve CloudFormation stack deployment failures across LZA-managed accounts"
context: "LZA deploys CloudFormation stacks across multiple accounts and regions. Stack failures occur during resource creation or update and indicate resource-level issues (permissions, quotas, conflicts). Stack names follow the pattern AWSAccelerator-<Phase>-<Account>-<Region>."
---

## Phase 1 — Triage

MUST:
- List failed stacks: `aws cloudformation list-stacks --stack-status-filter CREATE_FAILED UPDATE_FAILED ROLLBACK_COMPLETE UPDATE_ROLLBACK_COMPLETE --query 'StackSummaries[?contains(StackName,`AWSAccelerator`)].{Name:StackName,Status:StackStatus}'`
- Get stack events for the failed stack: `aws cloudformation describe-stack-events --stack-name <stack-name> --max-items 30 --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`||ResourceStatus==`UPDATE_FAILED`].{Resource:LogicalResourceId,Status:ResourceStatus,Reason:ResourceStatusReason}'`
- Identify the specific resource that failed and the error reason

SHOULD:
- Check if the failure is in the management account or a target account
- Verify IAM permissions for the CloudFormation execution role in the target account
- Check for resource quota limits in the target account

MAY:
- Check for resource naming conflicts with existing resources
- Review CloudFormation drift for the stack
- Check CloudTrail for the specific API call that failed

## Phase 2 — Remediate

MUST:
- Fix the resource-level error identified in stack events
- For permission errors: update the CloudFormation execution role
- For quota errors: request quota increases in the target account

SHOULD:
- Fix the LZA configuration that generated the failing resource
- Re-run the pipeline after fixing the root cause
- For ROLLBACK_COMPLETE stacks: delete and let LZA recreate them

MAY:
- Use CloudFormation drift detection to identify manual changes
- Check for resource dependencies that may need to be created first

## Safety Ratings

```
safety_ratings:
  - "cloudformation list-stacks: GREEN — read-only stack listing"
  - "cloudformation describe-stack-events: GREEN — read-only event history"
  - "cloudformation delete-stack: RED — deletes stack and its resources"
  - "cloudformation detect-stack-drift: GREEN — read-only drift check"
```

## Escalation Conditions

- Stack failure affects critical infrastructure (VPCs, IAM roles, security services)
- Multiple stacks fail across different accounts simultaneously
- Stack is in ROLLBACK_COMPLETE and contains resources that cannot be recreated
- Stack failure is caused by AWS service limits that cannot be increased

## Data Sensitivity

- CloudFormation stack events: MEDIUM — reveal resource creation details
- Stack resources: HIGH — contain infrastructure configuration
- Stack outputs: MEDIUM — may contain resource ARNs and endpoints
- CloudFormation execution role: HIGH — broad permissions for resource management

## Prohibited Actions

- NEVER delete LZA CloudFormation stacks without understanding what resources they manage
- NEVER manually update LZA-managed stacks — changes will be overwritten
- NEVER ignore ROLLBACK_COMPLETE stacks — they block future deployments
- NEVER modify CloudFormation execution role permissions without understanding the impact

## Phase 3 — Rollback

1. For UPDATE_ROLLBACK_COMPLETE stacks: fix the configuration and re-run the pipeline
2. For ROLLBACK_COMPLETE stacks: delete the stack and re-run the pipeline to recreate
3. If stack deletion fails due to resource dependencies, identify and remove dependencies first
4. Document all stack failures and resolutions
5. Verify all stacks are in a healthy state after recovery

## Common Issues

- symptoms: "Stack fails with 'Resource limit exceeded'"
  diagnosis: "AWS service quota reached in the target account for the specific resource type."
  resolution: "Check quotas: `aws service-quotas list-service-quotas --service-code <service>`. Request increase."

- symptoms: "Stack fails with 'Resource already exists'"
  diagnosis: "A resource with the same name/ID was created outside of LZA."
  resolution: "Remove the conflicting resource or import it into the stack. Update LZA config to avoid conflicts."

## Output Format

```yaml
root_cause: "cloudformation_stack — <specific_cause>"
evidence:
  - type: stack_error
    content: "<stack name, failed resource, and error reason>"
severity: HIGH
mitigation:
  immediate: "Fix resource-level error"
  long_term: "Implement pre-deployment resource validation"
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
