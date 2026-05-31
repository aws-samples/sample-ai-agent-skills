---
title: "C2 — ENI Limits (EC2 Launch Type)"
description: "Diagnose task placement failures caused by ENI limits on EC2 instances"
status: active
severity: HIGH
triggers:
  - "Cannot place task"
  - "ENI limit reached"
  - "No available network interfaces"
owner: devops-agent
objective: "Resolve ENI exhaustion and restore task placement"
context: "awsvpc network mode gives each task its own ENI. EC2 instances have ENI limits based on instance type. ENI trunking (account setting) increases the limit significantly. Without trunking, a t3.medium supports only 3 ENIs (1 primary + 2 tasks)."
---

## Phase 1 — Triage

MUST:
- Check container instance remaining resources: `aws ecs describe-container-instances --cluster <cluster> --container-instances <arns>` → remainingResources
- Count ENIs on the instance: `aws ec2 describe-network-interfaces --filters Name=attachment.instance-id,Values=<instance-id>`
- Check instance type ENI limit
- Verify if ENI trunking is enabled: `aws ecs list-account-settings --effective-settings true --name awsvpcTrunking`

SHOULD:
- Check if tasks are using awsvpc mode (only mode that consumes ENIs per task)
- Verify if bridge or host mode could be used instead
- Check if the cluster has enough instances

MAY:
- Enable ENI trunking to increase per-instance task density
- Consider using larger instance types with higher ENI limits

## Common Issues

- symptoms: "Task can't be placed, instance has CPU/memory but no ENIs"
  diagnosis: "ENI limit reached on the instance. awsvpc mode requires one ENI per task."
  resolution: "Enable ENI trunking, add more instances, or use bridge networking mode."

- symptoms: "ENI trunking enabled but not working"
  diagnosis: "ENI trunking requires supported instance types and ECS agent 1.28.0+."
  resolution: "Verify instance type supports trunking and update ECS agent."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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
3. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
4. If capacity provider strategy was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --capacity-provider-strategy <original>`
## Output Format

```yaml
root_cause: "eni_limits — <detail>"
evidence:
  - type: container_instance
    content: "<ENI count, instance type, trunking status>"
severity: HIGH
mitigation:
  immediate: "Enable ENI trunking or add more instances"
  long_term: "Use ENI trunking, right-size instances, consider Fargate for variable workloads"
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
