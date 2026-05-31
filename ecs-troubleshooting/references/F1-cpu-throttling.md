---
title: "F1 — CPU Throttling"
description: "Diagnose ECS task performance issues caused by CPU throttling"
status: active
severity: MEDIUM
triggers:
  - "Slow response times"
  - "CPU throttling"
  - "High CPU utilization"
owner: devops-agent
objective: "Identify CPU throttling and optimize CPU allocation"
context: "ECS CPU units: 1024 = 1 vCPU. Fargate: CPU is a hard limit. EC2: cpu is a soft limit (can burst if host has spare CPU) unless you set the hard limit. CPU throttling causes latency spikes."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch CPUUtilization metric for the service/task
- Get CPU allocation from task definition: cpu (task-level) and containerDefinitions[].cpu
- For Fargate: check if CPU is consistently at 100% (hard limit)
- For EC2: check container instance CPU availability

SHOULD:
- Correlate CPU spikes with latency increases
- Check if the issue is specific to certain request types
- Verify if the CPU allocation matches the workload requirements

MAY:
- Profile the application to identify CPU-intensive operations
- Check if garbage collection (Java) or similar runtime overhead is contributing

## Common Issues

- symptoms: "Fargate task at 100% CPU, latency spikes"
  diagnosis: "CPU allocation too low for the workload. Fargate enforces hard CPU limits."
  resolution: "Increase Fargate CPU allocation (256, 512, 1024, 2048, 4096)."

- symptoms: "EC2 task CPU high but host has spare CPU"
  diagnosis: "Container CPU hard limit set. Remove hard limit to allow bursting on EC2."
  resolution: "Use cpu as soft limit (reservation) on EC2, remove hard limit."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
## Output Format

```yaml
root_cause: "cpu_throttling — <detail>"
evidence:
  - type: cloudwatch_metric
    content: "<CPU utilization and allocation>"
severity: MEDIUM
mitigation:
  immediate: "Increase CPU allocation"
  long_term: "Right-size CPU based on profiling, use horizontal scaling"
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
