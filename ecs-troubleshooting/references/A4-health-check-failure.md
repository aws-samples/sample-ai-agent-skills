---
title: "A4 — Container Health Check Failure"
description: "Diagnose task failures caused by ECS container health check failures"
status: active
severity: HIGH
triggers:
  - "UNHEALTHY"
  - "Health check failed"
  - "Container health check"
owner: devops-agent
objective: "Fix container health check failures"
context: "ECS container health checks (defined in task definition) are different from ELB health checks. Container health check failure marks the task UNHEALTHY. If the container is essential, the task is stopped and replaced."
---

## Phase 1 — Triage

MUST:
- Check task health status: describe-tasks → containers[].healthStatus
- Get health check configuration from task definition: healthCheck.command, interval, timeout, retries, startPeriod
- Test the health check command manually (via ECS Exec if possible)
- Check container logs around the time of health check failure

SHOULD:
- Verify the health check endpoint is correct and responding
- Check if startPeriod is long enough for the application to initialize
- Verify the health check command is available in the container (curl, wget, etc.)

MAY:
- Distinguish between ECS container health check and ELB target health check
- Check if the application has a dedicated health endpoint

## Common Issues

- symptoms: "UNHEALTHY immediately after start"
  diagnosis: "startPeriod too short. Application hasn't finished initializing."
  resolution: "Increase startPeriod to allow for application startup time."

- symptoms: "Health check command fails: curl not found"
  diagnosis: "Health check uses curl but it's not installed in the container image."
  resolution: "Install curl in the image, or use a different health check method (e.g., CMD-SHELL with wget or /bin/sh)."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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
2. If health check settings were modified, restore original target group health check configuration
3. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "health_check_failure — <detail>"
evidence:
  - type: ecs_task
    content: "<health status and check configuration>"
severity: HIGH
mitigation:
  immediate: "Fix health check command or increase startPeriod"
  long_term: "Implement proper health endpoints, tune check parameters"
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
