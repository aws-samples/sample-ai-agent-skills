---
title: "E1 — Stuck Deployment"
description: "Diagnose ECS service deployments that are stuck and not completing"
status: active
severity: HIGH
triggers:
  - "Deployment stuck"
  - "Service not reaching desired count"
  - "Rolling update not completing"
owner: devops-agent
objective: "Unblock the stuck deployment and restore service stability"
context: "ECS rolling deployments replace tasks gradually. A deployment gets stuck when new tasks fail to start or fail health checks. The service won't drain old tasks until new tasks are healthy. Check minimumHealthyPercent and maximumPercent settings."
---

## Phase 1 — Triage

MUST:
- Check service deployments: `aws ecs describe-services --cluster <cluster> --services <service>` → deployments[]
- Look for multiple active deployments (PRIMARY + ACTIVE = stuck)
- Check service events for error messages
- Check if new tasks are starting and failing (describe stopped tasks)

SHOULD:
- Check load balancer health check status for new tasks
- Verify the new task definition is valid (image exists, ports correct)
- Check if there's enough capacity (CPU, memory, ENIs) for new tasks

MAY:
- Check deployment configuration: minimumHealthyPercent, maximumPercent
- Verify deployment circuit breaker settings
- Check if the deployment controller is ECS or CODE_DEPLOY

## Common Issues

- symptoms: "New tasks start but fail health checks, old tasks stay running"
  diagnosis: "New task definition has a bug or health check misconfiguration."
  resolution: "Fix the task definition, or roll back by updating service to previous task definition."

- symptoms: "Can't place new tasks, no capacity"
  diagnosis: "Cluster doesn't have enough CPU/memory/ENIs for both old and new tasks during rolling update."
  resolution: "Add capacity, reduce minimumHealthyPercent, or use a smaller maximumPercent."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
3. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
4. If health check settings were modified, restore original target group health check configuration
## Output Format

```yaml
root_cause: "stuck_deployment — <detail>"
evidence:
  - type: ecs_service
    content: "<deployment state and service events>"
severity: HIGH
mitigation:
  immediate: "Roll back to previous task definition or fix the new one"
  long_term: "Enable deployment circuit breaker, use canary deployments"
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
