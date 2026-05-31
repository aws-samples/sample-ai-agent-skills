---
title: "H3 — Resource Reservation Issues"
description: "Diagnose task placement failures caused by resource reservation mismatches"
status: active
severity: MEDIUM
triggers:
  - "Cannot place task"
  - "Resource mismatch"
  - "CPU/memory reservation"
owner: devops-agent
objective: "Fix resource reservation configuration for optimal task placement"
context: "EC2 launch type: cpu and memory in task definition are reservations (soft limits) for scheduling. Fargate: they are hard limits. Over-reserving wastes capacity. Under-reserving causes contention."
---

## Phase 1 — Triage

MUST:
- Check task definition CPU and memory settings (task-level and container-level)
- For EC2: compare reservations with actual usage
- For Fargate: verify CPU/memory combination is valid
- Check cluster resource utilization

SHOULD:
- Check if containers are using more than reserved (EC2 allows this)
- Verify Fargate CPU/memory combinations are valid (not all combinations work)
- Check if task-level and container-level resources are consistent

MAY:
- Analyze CloudWatch metrics to right-size reservations
- Check if bin-packing placement strategy could improve utilization

## Common Issues

- symptoms: "Tasks can't be placed but instances show low actual CPU usage"
  diagnosis: "CPU reservations are too high relative to actual usage. Scheduling sees no room."
  resolution: "Reduce CPU reservation to match actual usage patterns."

- symptoms: "Invalid Fargate CPU/memory combination"
  diagnosis: "Fargate only supports specific CPU/memory combinations (e.g., 256 CPU requires 512-2048 memory)."
  resolution: "Use a valid Fargate CPU/memory combination."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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
2. If capacity provider strategy was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --capacity-provider-strategy <original>`
## Output Format

```yaml
root_cause: "resource_reservation — <detail>"
evidence:
  - type: task_definition
    content: "<CPU/memory settings and actual usage>"
severity: MEDIUM
mitigation:
  immediate: "Adjust reservations to match actual usage"
  long_term: "Monitor and right-size regularly, use bin-packing placement strategy"
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
