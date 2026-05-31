---
title: "G3 — Deregistration Delay Issues"
description: "Diagnose issues caused by target deregistration delay during deployments"
status: active
severity: MEDIUM
triggers:
  - "Slow deployments"
  - "Connection errors during deployment"
  - "Deregistration delay"
owner: devops-agent
objective: "Optimize deregistration delay for smooth deployments"
context: "When a task is stopped, the target is deregistered from the target group. Deregistration delay (default 300s) keeps the target draining for in-flight requests. Too long = slow deployments. Too short = dropped connections."
---

## Phase 1 — Triage

MUST:
- Check deregistration delay: `aws elbv2 describe-target-group-attributes --target-group-arn <arn>` → deregistration_delay.timeout_seconds
- Check ECS service stopTimeout setting
- Verify deployment speed expectations vs actual deployment time

SHOULD:
- Check if connections are being dropped during deployments (5xx errors)
- Verify the application handles SIGTERM gracefully
- Check if the deregistration delay is longer than the stopTimeout

MAY:
- Monitor ALB metrics during deployments for error spikes
- Test with different deregistration delay values

## Common Issues

- symptoms: "Deployments take 10+ minutes, deregistration delay is 300s"
  diagnosis: "Default 300s deregistration delay adds 5 minutes per batch of tasks."
  resolution: "Reduce deregistration delay to 30-60s if the application handles graceful shutdown."

- symptoms: "502 errors during deployment"
  diagnosis: "Deregistration delay too short, in-flight requests are dropped."
  resolution: "Increase deregistration delay, ensure application handles SIGTERM and drains connections."


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

1. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
2. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
3. If health check settings were modified, restore original target group health check configuration
## Output Format

```yaml
root_cause: "deregistration_delay — <too long|too short>"
evidence:
  - type: target_group
    content: "<deregistration delay setting and deployment behavior>"
severity: MEDIUM
mitigation:
  immediate: "Adjust deregistration delay to match application needs"
  long_term: "Implement graceful shutdown, tune delay based on request duration"
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
