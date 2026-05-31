---
title: "A2 — OOM Kill"
description: "Diagnose task failures caused by out-of-memory kills"
status: active
severity: HIGH
triggers:
  - "OutOfMemoryError"
  - "Exit code 137"
  - "Container killed"
  - "OOM"
owner: devops-agent
objective: "Identify memory exhaustion cause and right-size container memory"
context: "ECS has hard limit (memory) and soft limit (memoryReservation). Hard limit = OOM kill threshold. Soft limit = scheduling reservation only. Exit code 137 = SIGKILL (OOM or manual). Check stoppedReason for 'OutOfMemoryError' to confirm OOM."
---

## Phase 1 — Triage

MUST:
- Check stopped task: exit code 137 AND stoppedReason contains "OutOfMemoryError"
- Get task definition memory limits: `aws ecs describe-task-definition --task-definition <family:revision>` → containerDefinitions[].memory, memoryReservation
- Check CloudWatch metrics: MemoryUtilization for the service
- Check container logs for memory-related errors before the kill

SHOULD:
- Determine if memory usage is growing over time (leak) or spiking (burst)
- Check if the issue started after a code deployment
- Verify if Fargate task-level memory is sufficient (shared across containers)

MAY:
- Profile the application's memory usage
- Check if JVM heap settings (Java) or similar runtime limits are misconfigured

## Common Issues

- symptoms: "Memory grows steadily until OOM kill"
  diagnosis: "Memory leak in application code."
  resolution: "Fix the memory leak. As a temporary measure, increase memory limit."

- symptoms: "OOM on specific requests or workloads"
  diagnosis: "Memory-intensive operation exceeds container limit."
  resolution: "Increase memory limit or optimize the memory-intensive operation."

- symptoms: "Fargate task OOM but individual container limits seem fine"
  diagnosis: "Total memory across all containers exceeds Fargate task memory."
  resolution: "Increase Fargate task-level memory or reduce individual container limits."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
3. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
4. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "oom_kill — <memory leak|burst|undersized>"
evidence:
  - type: ecs_task
    content: "<exit code, stoppedReason, memory metrics>"
severity: HIGH
mitigation:
  immediate: "Increase memory limit"
  long_term: "Fix memory leak, implement memory monitoring and alerts"
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
