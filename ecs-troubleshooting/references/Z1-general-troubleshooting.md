---
title: "Z1 — General ECS Troubleshooting"
description: "Catch-all runbook for ECS issues that don't match a specific category"
status: active
severity: MEDIUM
triggers:
  - "ECS issue"
  - "Something wrong with ECS"
  - "Container problem"
owner: devops-agent
objective: "Systematically investigate unclassified ECS issues"
context: "Use this runbook when the issue doesn't clearly match a specific category. Follow the systematic approach to narrow down the problem domain."
---

## Phase 1 — Triage

MUST:
- Check service state: `aws ecs describe-services --cluster <cluster> --services <service>`
- Check service events (last 100) for error messages
- List stopped tasks and check stoppedReason and exit codes
- Check running tasks health status
- Check container logs for errors

SHOULD:
- Check task definition for obvious issues (image, ports, memory, CPU)
- Verify cluster has capacity (EC2 instances or Fargate)
- Check load balancer health if applicable
- Review recent deployments or configuration changes

MAY:
- Check CloudTrail for recent API changes
- Check CloudWatch metrics for anomalies
- Use ECS Exec to debug a running container

## Systematic Approach

1. Service level: Is the service reaching desired count? Check events.
2. Task level: Are tasks starting? Check stopped tasks for reasons.
3. Container level: Are containers healthy? Check exit codes and logs.
4. Network level: Can tasks communicate? Check SG, NACL, routing.
5. IAM level: Are permissions correct? Check execution role and task role.
6. Capacity level: Is there room? Check instance resources or Fargate availability.


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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

1. If task definition was updated, roll back to previous revision: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <previous-revision>`
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
3. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
4. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
5. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
6. If health check settings were modified, restore original target group health check configuration
7. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
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
