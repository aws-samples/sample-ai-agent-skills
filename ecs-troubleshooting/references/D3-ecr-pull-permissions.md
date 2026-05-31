---
title: "D3 — ECR Pull Permissions"
description: "Diagnose ECR image pull failures caused by IAM or repository policy issues"
status: active
severity: HIGH
triggers:
  - "CannotPullContainerError"
  - "denied: Your authorization token has expired"
  - "ECR pull access denied"
owner: devops-agent
objective: "Fix ECR pull permissions"
context: "ECR pull requires: execution role with ecr:GetAuthorizationToken (account-level) + ecr:BatchGetImage + ecr:GetDownloadUrlForLayer (repository-level). Cross-account pulls also need ECR repository policy."
---

## Phase 1 — Triage

MUST:
- Verify execution role has ecr:GetAuthorizationToken (not scoped to a specific repo)
- Verify execution role has ecr:BatchGetImage and ecr:GetDownloadUrlForLayer
- For cross-account: check ECR repository policy allows the execution role
- Verify the ECR repository exists and image tag exists

SHOULD:
- Check if the repository has a restrictive policy
- Verify the execution role ARN is correct in the task definition
- Check if ECR is in a different region (requires cross-region pull)

MAY:
- Test ECR auth: `aws ecr get-authorization-token`
- Check if ECR replication could simplify cross-account/region access

## Common Issues

- symptoms: "Pull works for some repos but not others"
  diagnosis: "Repository policy on the failing repo restricts access."
  resolution: "Update ECR repository policy to allow the execution role."

- symptoms: "Authorization token expired"
  diagnosis: "ECR auth tokens expire after 12 hours. ECS agent should auto-refresh."
  resolution: "Verify ECS agent is up to date. Check execution role has ecr:GetAuthorizationToken."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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
## Output Format

```yaml
root_cause: "ecr_pull_permissions — <detail>"
evidence:
  - type: iam
    content: "<execution role permissions and ECR repository policy>"
severity: HIGH
mitigation:
  immediate: "Fix execution role or repository policy"
  long_term: "Use AmazonECSTaskExecutionRolePolicy, set up ECR replication for cross-account"
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
