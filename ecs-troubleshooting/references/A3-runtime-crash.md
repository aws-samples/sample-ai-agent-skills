---
title: "A3 — Container Runtime Crash"
description: "Diagnose container crashes caused by runtime errors, segfaults, or startup failures"
status: active
severity: HIGH
triggers:
  - "Exit code 139"
  - "Segmentation fault"
  - "Container failed to start"
  - "Runtime error"
owner: devops-agent
objective: "Identify the runtime crash cause and restore container stability"
context: "Exit code 139 = SIGSEGV (segmentation fault). Common with native code, corrupted binaries, or incompatible libraries. Exit code 126 = permission denied. Exit code 127 = command not found."
---

## Phase 1 — Triage

MUST:
- Check exit code and stoppedReason from describe-tasks
- Check container logs for crash output, stack traces, or core dumps
- Verify the container image exists and is pullable
- Check if the entrypoint/command in task definition is correct

SHOULD:
- Verify the image architecture matches the task (x86_64 vs arm64)
- Check if shared libraries are present in the container image
- Review recent image changes or rebuilds

MAY:
- Run the container locally to reproduce the crash
- Check if the crash is intermittent or consistent

## Common Issues

- symptoms: "Exit code 139, native library crash"
  diagnosis: "Segmentation fault in native code. Incompatible library or corrupted binary."
  resolution: "Rebuild the image, verify library compatibility, check architecture match."

- symptoms: "Exit code 127, command not found"
  diagnosis: "Entrypoint or command binary doesn't exist in the container image."
  resolution: "Fix the entrypoint/command in task definition or rebuild the image."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
2. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
3. If health check settings were modified, restore original target group health check configuration
4. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "runtime_crash — <exit code and cause>"
evidence:
  - type: ecs_task
    content: "<exit code, crash output, image details>"
severity: HIGH
mitigation:
  immediate: "Fix the crashing binary or configuration"
  long_term: "Add container health checks, test images before deployment"
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
