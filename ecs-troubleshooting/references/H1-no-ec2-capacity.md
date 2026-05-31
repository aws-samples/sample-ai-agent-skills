---
title: "H1 — No EC2 Capacity"
description: "Diagnose task placement failures due to insufficient EC2 container instance capacity"
status: active
severity: HIGH
triggers:
  - "service was unable to place a task"
  - "No container instances"
  - "Insufficient CPU or memory"
owner: devops-agent
objective: "Restore EC2 capacity for ECS task placement"
context: "EC2 launch type requires container instances with available CPU, memory, ports, and ENIs. Tasks can't be placed if no instance meets all requirements. Check remaining resources on each instance."
---

## Phase 1 — Triage

MUST:
- Check container instances: `aws ecs describe-container-instances --cluster <cluster> --container-instances <arns>` → remainingResources
- Compare task CPU/memory requirements with available resources
- Check if instances are ACTIVE and connected (agentConnected=true)
- Verify the cluster has registered container instances

SHOULD:
- Check if Auto Scaling group can scale up
- Verify instance type has enough CPU/memory for the task
- Check if port conflicts prevent placement (host/bridge mode)

MAY:
- Check if placement constraints or strategies are too restrictive
- Verify if the ECS agent is up to date

## Common Issues

- symptoms: "No container instances in cluster"
  diagnosis: "ASG not launched, instances not joining cluster, or wrong cluster name."
  resolution: "Check ASG, verify ECS_CLUSTER in user data, check instance IAM role."

- symptoms: "Instances exist but not enough CPU/memory"
  diagnosis: "Existing tasks consuming all resources. Need more or larger instances."
  resolution: "Scale up ASG, use larger instance types, or optimize task resource requests."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
3. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
4. If capacity provider strategy was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --capacity-provider-strategy <original>`
## Output Format

```yaml
root_cause: "no_ec2_capacity — <detail>"
evidence:
  - type: container_instance
    content: "<remaining resources and task requirements>"
severity: HIGH
mitigation:
  immediate: "Scale up ASG or reduce task resource requirements"
  long_term: "Use capacity providers with managed scaling, consider Fargate"
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
