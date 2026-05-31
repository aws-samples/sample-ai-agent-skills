---
title: "F2 — Memory Pressure"
description: "Diagnose ECS task performance issues caused by memory pressure"
status: active
severity: HIGH
triggers:
  - "High memory utilization"
  - "Approaching OOM"
  - "Memory pressure"
  - "Swap usage"
owner: devops-agent
objective: "Identify memory pressure and optimize memory allocation"
context: "ECS memory: hard limit (memory) = OOM kill threshold. Soft limit (memoryReservation) = scheduling only. Fargate: task-level memory is shared across containers. Memory pressure causes GC thrashing, swap usage, and eventual OOM."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch MemoryUtilization metric for the service
- Get memory allocation from task definition: memory and memoryReservation
- Check if memory usage is growing over time (leak) or stable but high
- Check for OOM events in stopped tasks

SHOULD:
- Check if the application has memory-related configuration (JVM heap, Node.js max-old-space-size)
- Verify if Fargate task-level memory is sufficient for all containers
- Check if sidecar containers are consuming unexpected memory

MAY:
- Profile memory usage to identify leaks
- Check if the workload has changed (more data, more connections)

## Common Issues

- symptoms: "Memory grows steadily, eventually OOM killed"
  diagnosis: "Memory leak in application code."
  resolution: "Fix the memory leak. Increase memory as temporary mitigation."

- symptoms: "Memory at 90% but stable, occasional OOM on traffic spikes"
  diagnosis: "Memory allocation too close to baseline usage. No headroom for spikes."
  resolution: "Increase memory allocation to provide headroom (aim for 70% average utilization)."


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
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
## Output Format

```yaml
root_cause: "memory_pressure — <leak|undersized|spike>"
evidence:
  - type: cloudwatch_metric
    content: "<memory utilization trend and allocation>"
severity: HIGH
mitigation:
  immediate: "Increase memory allocation"
  long_term: "Fix memory leaks, implement memory monitoring and alerts"
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
