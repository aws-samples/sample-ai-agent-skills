---
title: "I1 — ECS Exec Failures"
description: "Diagnose failures when using ECS Exec to connect to running containers"
status: active
severity: MEDIUM
triggers:
  - "ECS Exec failed"
  - "execute-command error"
  - "SSM session failed"
owner: devops-agent
objective: "Fix ECS Exec configuration and restore container debugging access"
context: "ECS Exec uses SSM (Systems Manager) to open a session to a running container. Requirements: enableExecuteCommand=true on service, task role with ssmmessages permissions, Fargate platform 1.4.0+, SSM plugin installed locally."
---

## Phase 1 — Triage

MUST:
- Verify enableExecuteCommand is true on the service: `aws ecs describe-services --cluster <cluster> --services <service>` → enableExecuteCommand
- Check task role has ssmmessages permissions (ssmmessages:CreateControlChannel, ssmmessages:CreateDataChannel, ssmmessages:OpenControlChannel, ssmmessages:OpenDataChannel)
- For Fargate: verify platform version is 1.4.0 or later
- Verify SSM Session Manager plugin is installed locally

SHOULD:
- Check if the container has /bin/sh or the specified command available
- Verify VPC endpoints for SSM if in a private subnet (ssmmessages endpoint)
- Check if the task is in RUNNING state

MAY:
- Check CloudTrail for SSM session errors
- Verify the container is not in a crash loop

## Common Issues

- symptoms: "execute-command returns 'The execute command failed'"
  diagnosis: "enableExecuteCommand not set on the service, or task role missing ssmmessages permissions."
  resolution: "Update service with --enable-execute-command, add ssmmessages permissions to task role."

- symptoms: "Session starts but immediately closes"
  diagnosis: "Container doesn't have the specified shell (/bin/sh or /bin/bash)."
  resolution: "Use a shell that exists in the container image, or install one."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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
## Output Format

```yaml
root_cause: "ecs_exec — <detail>"
evidence:
  - type: ecs_service
    content: "<exec configuration and error message>"
severity: MEDIUM
mitigation:
  immediate: "Enable execute command and fix permissions"
  long_term: "Include debugging tools in container images, standardize exec configuration"
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
