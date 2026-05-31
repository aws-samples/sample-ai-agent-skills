---
title: "F3 — I/O Bottleneck"
description: "Diagnose ECS task performance issues caused by I/O bottlenecks"
status: active
severity: MEDIUM
triggers:
  - "Slow disk operations"
  - "High I/O wait"
  - "EFS latency"
  - "Ephemeral storage full"
owner: devops-agent
objective: "Identify I/O bottlenecks and optimize storage performance"
context: "Fargate ephemeral storage: 20 GB default, up to 200 GB configurable. EFS mounts add latency. EC2 launch type uses host storage. I/O bottlenecks cause high latency and timeouts."
---

## Phase 1 — Triage

MUST:
- Check if the task uses EFS volumes (higher latency than local storage)
- For Fargate: check ephemeral storage configuration and usage
- Check application logs for I/O-related errors or timeouts
- Check CloudWatch metrics for EFS (if used): PercentIOLimit, BurstCreditBalance

SHOULD:
- Determine if the I/O pattern is read-heavy, write-heavy, or mixed
- Check if EFS throughput mode is appropriate (bursting vs provisioned)
- Verify ephemeral storage isn't full

MAY:
- Profile I/O operations in the application
- Check if caching could reduce I/O load

## Common Issues

- symptoms: "High latency on file operations, using EFS"
  diagnosis: "EFS has higher latency than local storage. Burst credits may be depleted."
  resolution: "Switch to provisioned throughput, use caching, or move hot data to ephemeral storage."

- symptoms: "Fargate task failing, ephemeral storage full"
  diagnosis: "Application writing too much data to local storage (logs, temp files)."
  resolution: "Increase ephemeral storage (up to 200 GB), clean up temp files, stream logs to CloudWatch."


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

1. If log configuration was changed, create a new task definition revision with the original log driver settings
2. Document all changes made during troubleshooting for audit trail
3. Verify service health after any rollback using monitoring dashboards
## Output Format

```yaml
root_cause: "io_bottleneck — <detail>"
evidence:
  - type: storage
    content: "<I/O metrics and storage configuration>"
severity: MEDIUM
mitigation:
  immediate: "Increase storage or switch throughput mode"
  long_term: "Optimize I/O patterns, use caching, right-size storage"
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
