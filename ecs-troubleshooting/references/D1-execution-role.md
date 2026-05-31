---
title: "D1 — Execution Role Issues"
description: "Diagnose task failures caused by missing or misconfigured execution role permissions"
status: active
severity: HIGH
triggers:
  - "AccessDeniedException"
  - "Unable to pull secrets"
  - "CannotPullContainerError"
  - "Log driver error"
owner: devops-agent
objective: "Fix execution role permissions to restore task operations"
context: "Execution role (executionRoleArn) is used by the ECS AGENT for: pulling images from ECR, pushing logs to CloudWatch, pulling secrets from Secrets Manager/SSM, and pulling configs. This is NOT the task role."
---

## Phase 1 — Triage

MUST:
- Check execution role ARN in task definition
- List role policies: `aws iam list-attached-role-policies --role-name <role>`
- Verify ECR permissions: ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer
- Verify CloudWatch Logs permissions: logs:CreateLogStream, logs:PutLogEvents

SHOULD:
- Check if secrets are referenced in task definition (requires secretsmanager:GetSecretValue or ssm:GetParameters)
- Verify the role trust policy allows ecs-tasks.amazonaws.com to assume it
- Check if KMS permissions are needed for encrypted secrets

MAY:
- Compare with AmazonECSTaskExecutionRolePolicy managed policy
- Check CloudTrail for AccessDenied events from the execution role

## Common Issues

- symptoms: "CannotPullContainerError with auth failure"
  diagnosis: "Execution role missing ECR permissions."
  resolution: "Attach AmazonECSTaskExecutionRolePolicy or add ECR permissions."

- symptoms: "Task starts but no logs appear in CloudWatch"
  diagnosis: "Execution role missing CloudWatch Logs permissions."
  resolution: "Add logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents."

- symptoms: "ResourceInitializationError: unable to pull secrets"
  diagnosis: "Execution role missing secretsmanager:GetSecretValue or ssm:GetParameters."
  resolution: "Add the required secrets/SSM permissions to the execution role."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- Service handles production traffic or is business-critical
- Fix requires modifying the task definition for a running service
- Resolution involves changing capacity provider or cluster configuration
- Multiple services affected suggesting cluster-level or account-level issue
- Deployment rollback has been triggered by the circuit breaker

## Data Sensitivity

HIGH - ECS task definitions may contain secrets ARNs, environment variables with credentials, and sensitive configuration. Container logs may capture application data including PII. Service Connect and service discovery configurations may expose internal architecture.

## Prohibited Actions

- NEVER suggest stopping all tasks in a service to fix a deployment issue - this causes a full outage
- NEVER suggest removing or modifying the task execution role while tasks are running
- NEVER recommend deleting a service without first scaling desired count to 0 and draining connections
- NEVER suggest modifying a running task definition - create a new revision instead
- NEVER recommend disabling the deployment circuit breaker on a production service

## Phase 3 - Rollback

1. If task definition was updated, roll back to previous revision: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <previous-revision>`
2. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
3. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "execution_role — <missing permission>"
evidence:
  - type: iam
    content: "<role ARN and missing permissions>"
severity: HIGH
mitigation:
  immediate: "Add required permissions to execution role"
  long_term: "Use AmazonECSTaskExecutionRolePolicy as baseline, add specific permissions as needed"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
