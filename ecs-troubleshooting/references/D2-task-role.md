---
title: "D2 — Task Role Issues"
description: "Diagnose application-level permission failures caused by task role misconfiguration"
status: active
severity: HIGH
triggers:
  - "AccessDeniedException from application"
  - "Unable to assume role"
  - "Credentials not found"
owner: devops-agent
objective: "Fix task role permissions for application access to AWS services"
context: "Task role (taskRoleArn) provides AWS credentials to the APPLICATION running inside the container. It's used for application-level API calls (DynamoDB, S3, SQS, etc.). This is NOT the execution role."
---

## Phase 1 — Triage

MUST:
- Check task role ARN in task definition (taskRoleArn)
- List role policies: `aws iam list-attached-role-policies --role-name <role>`
- Verify the role has permissions for the AWS services the application uses
- Check the role trust policy allows ecs-tasks.amazonaws.com

SHOULD:
- Check container logs for specific AccessDeniedException messages
- Verify the application is using the task role credentials (not hardcoded credentials)
- Check if the ECS credential endpoint is accessible from the container

MAY:
- Test permissions using the AWS CLI with the task role
- Check if the application uses a specific SDK credential chain

## Common Issues

- symptoms: "Application gets AccessDeniedException for DynamoDB"
  diagnosis: "Task role missing DynamoDB permissions."
  resolution: "Add required DynamoDB permissions to the task role."

- symptoms: "Credentials not found in container"
  diagnosis: "taskRoleArn not set in task definition, or ECS credential endpoint blocked."
  resolution: "Set taskRoleArn in task definition. For bridge mode, verify ECS_CONTAINER_METADATA_URI is set."


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
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
3. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
4. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "task_role — <missing permission>"
evidence:
  - type: iam
    content: "<role ARN and missing permissions>"
severity: HIGH
mitigation:
  immediate: "Add required permissions to task role"
  long_term: "Use least-privilege policies, separate task roles per service"
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
