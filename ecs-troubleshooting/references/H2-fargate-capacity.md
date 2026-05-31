---
title: "H2 — Fargate Capacity Issues"
description: "Diagnose task placement failures on Fargate"
status: active
severity: HIGH
triggers:
  - "Fargate capacity unavailable"
  - "PROVISIONING timeout"
  - "Task stuck in PROVISIONING"
owner: devops-agent
objective: "Resolve Fargate capacity issues and restore task placement"
context: "Fargate capacity is managed by AWS but can be temporarily unavailable in specific AZs or for specific CPU/memory combinations. Fargate Spot has higher chance of capacity issues."
---

## Phase 1 — Triage

MUST:
- Check task status: is it stuck in PROVISIONING?
- Check service events for capacity-related messages
- Verify the CPU/memory combination is valid for Fargate
- Check if the issue is in a specific AZ

SHOULD:
- Try deploying in multiple AZs (use multiple subnets)
- Check if Fargate Spot is being used (higher interruption risk)
- Verify the platform version is valid

MAY:
- Check AWS Health Dashboard for Fargate service issues
- Try a different CPU/memory combination

## Common Issues

- symptoms: "Task stuck in PROVISIONING for minutes"
  diagnosis: "Temporary Fargate capacity shortage in the selected AZ."
  resolution: "Configure service to use multiple AZs. Retry usually succeeds."

- symptoms: "Fargate Spot tasks being interrupted"
  diagnosis: "Fargate Spot capacity reclaimed by AWS."
  resolution: "Use Fargate (non-Spot) for critical tasks, Spot for fault-tolerant workloads."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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

1. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
2. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
3. If capacity provider strategy was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --capacity-provider-strategy <original>`
## Output Format

```yaml
root_cause: "fargate_capacity — <detail>"
evidence:
  - type: ecs_task
    content: "<task state and provisioning details>"
severity: HIGH
mitigation:
  immediate: "Use multiple AZs, retry, or switch to non-Spot"
  long_term: "Configure multi-AZ subnets, use capacity provider strategy with Fargate + Fargate Spot"
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
