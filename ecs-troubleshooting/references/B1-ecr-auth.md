---
title: "B1 — ECR Authentication Failures"
description: "Diagnose image pull failures caused by ECR authentication issues"
status: active
severity: HIGH
triggers:
  - "CannotPullContainerError"
  - "pull access denied"
  - "no basic auth credentials"
  - "401 Unauthorized"
owner: devops-agent
objective: "Fix ECR authentication and restore image pulling"
context: "ECR authentication requires the EXECUTION ROLE (not task role) to have ecr:GetAuthorizationToken, ecr:BatchGetImage, and ecr:GetDownloadUrlForLayer permissions. ECR auth tokens expire after 12 hours."
---

## Phase 1 — Triage

MUST:
- Check stopped task stoppedReason for "CannotPullContainerError"
- Verify execution role has ECR permissions: `aws iam get-role-policy` or `aws iam list-attached-role-policies`
- Check if the ECR repository exists and the image tag exists
- Verify the execution role ARN in the task definition is correct

SHOULD:
- Check if the ECR repository is in a different account (cross-account requires repository policy)
- Verify the ECR repository policy allows the execution role
- Check if the image was recently deleted or the tag was overwritten

MAY:
- Test ECR auth manually: `aws ecr get-login-password`
- Check if a VPC endpoint for ECR is needed (Fargate in private subnet)

## Common Issues

- symptoms: "CannotPullContainerError: pull access denied"
  diagnosis: "Execution role missing ECR permissions."
  resolution: "Attach AmazonECSTaskExecutionRolePolicy or add ecr:GetAuthorizationToken, ecr:BatchGetImage, ecr:GetDownloadUrlForLayer."

- symptoms: "Cross-account ECR pull fails"
  diagnosis: "ECR repository policy doesn't allow the execution role from the other account."
  resolution: "Add a repository policy allowing the cross-account execution role."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
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
4. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
5. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "ecr_auth — <missing permission or policy>"
evidence:
  - type: iam
    content: "<execution role and missing permissions>"
severity: HIGH
mitigation:
  immediate: "Add required ECR permissions to execution role"
  long_term: "Use AmazonECSTaskExecutionRolePolicy, set up cross-account ECR replication"
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
