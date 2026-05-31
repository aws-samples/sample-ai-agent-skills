---
title: "G2 — Target Registration Issues"
description: "Diagnose ECS tasks not registering with load balancer target groups"
status: active
severity: HIGH
triggers:
  - "No targets registered"
  - "Task running but not in target group"
  - "Target registration failed"
owner: devops-agent
objective: "Fix target registration and restore load balancer traffic"
context: "ECS automatically registers tasks with target groups when using service load balancer configuration. Registration requires correct target group ARN, container name, and container port in the service definition."
---

## Phase 1 — Triage

MUST:
- Check service load balancer configuration: describe-services → loadBalancers[]
- Verify target group ARN, container name, and container port are correct
- Check if tasks are running: `aws ecs list-tasks --cluster <cluster> --service-name <service> --desired-status RUNNING`
- Check target group targets: `aws elbv2 describe-target-health --target-group-arn <arn>`

SHOULD:
- Verify the target group type matches (ip for awsvpc, instance for bridge/host)
- Check if the container port in the service matches the container port in the task definition
- Verify the target group is in the same VPC as the tasks

MAY:
- Check service events for registration errors
- Verify the ECS service-linked role has ELB permissions

## Common Issues

- symptoms: "Tasks running but not appearing in target group"
  diagnosis: "Target group type mismatch. awsvpc mode requires 'ip' target type, not 'instance'."
  resolution: "Recreate target group with correct target type (ip for awsvpc, instance for bridge)."

- symptoms: "Container port mismatch between service and task definition"
  diagnosis: "Service loadBalancers[].containerPort doesn't match the port in containerDefinitions[].portMappings."
  resolution: "Update service or task definition to use matching ports."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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
3. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
4. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
5. If health check settings were modified, restore original target group health check configuration
## Output Format

```yaml
root_cause: "target_registration — <detail>"
evidence:
  - type: target_group
    content: "<target group configuration and service load balancer settings>"
severity: HIGH
mitigation:
  immediate: "Fix target group type or port configuration"
  long_term: "Use IaC to ensure consistent target group and service configuration"
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
