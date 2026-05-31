---
title: "B1 — Task Placement Failures"
description: "Diagnose task placement failures on external instances"
status: active
severity: HIGH
triggers: ["task not placed", "no container instances", "placement failed", "insufficient resources"]
owner: devops-agent
objective: "Resolve task placement failures on ECS Anywhere external instances"
context: "Tasks must use EXTERNAL launch type. Placement depends on available resources, constraints, and instance connectivity."
---

## Phase 1 — Triage

MUST:
- Check service events for placement errors: `aws ecs describe-services --cluster <cluster-name> --services <service-name> --query 'services[*].events[:10]'`
- Check available instances and resources: `aws ecs describe-container-instances --cluster <cluster-name> --container-instances $(aws ecs list-container-instances --cluster <cluster-name> --status ACTIVE --query 'containerInstanceArns' --output text) --query 'containerInstances[*].{Id:ec2InstanceId,CPU:remainingResources[?name==\`CPU\`].integerValue|[0],Memory:remainingResources[?name==\`MEMORY\`].integerValue|[0],Agent:agentConnected}'`
- Verify task definition uses EXTERNAL launch type: `aws ecs describe-task-definition --task-definition <task-def> --query 'taskDefinition.requiresCompatibilities'`

SHOULD:
- Check if placement constraints match available instance attributes
- Verify task CPU/memory requirements fit within available instance resources
- Check if the network mode is compatible (host or bridge, NOT awsvpc)

MAY:
- Check for port conflicts in host network mode
- Review placement strategy configuration
- Check if instances are draining

## Phase 2 — Remediate

MUST:
- Ensure task definition includes EXTERNAL in requiresCompatibilities
- Verify sufficient CPU and memory on available instances
- Use host or bridge network mode (not awsvpc)

SHOULD:
- Register additional external instances if capacity is insufficient
- Adjust task CPU/memory requirements to fit available instances
- Remove overly restrictive placement constraints

MAY:
- Implement custom attributes on instances for targeted placement
- Set up capacity monitoring alerts

## Common Issues

- symptoms: "Service event: 'no container instances found matching constraints'"
  diagnosis: "No ACTIVE external instances with sufficient resources, or task definition missing EXTERNAL launch type."
  resolution: "Verify EXTERNAL in requiresCompatibilities. Check instance resources. Register more instances if needed."

- symptoms: "Task placed but immediately stops with 'CannotPullContainerError'"
  diagnosis: "External instance cannot reach ECR or Docker Hub to pull the container image."
  resolution: "Verify outbound HTTPS connectivity to ECR endpoints. Check Docker/containerd proxy settings."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-services: GREEN — read-only"
  - "ecs describe-container-instances: GREEN — read-only"
  - "ecs describe-task-definition: GREEN — read-only"
  - "ecs update-service: YELLOW — changes desired task count"
  - "ecs register-task-definition: YELLOW — creates new task definition version"
```

## Escalation Conditions

- No external instances available and new instances cannot be registered quickly
- Task requires more resources than any single external instance can provide
- Placement failures cause service outage
- Task requires awsvpc mode which is not supported on external instances

## Data Sensitivity

- `aws ecs describe-services`: MEDIUM — reveals service configuration and events
- `aws ecs describe-container-instances`: MEDIUM — reveals instance resources and attributes
- `aws ecs describe-task-definition`: MEDIUM — reveals container configuration and secrets references
- Service events: LOW — placement error messages

## Prohibited Actions

- NEVER force-place tasks on instances with insufficient resources
- NEVER use awsvpc network mode for EXTERNAL launch type tasks
- NEVER remove placement constraints without understanding their purpose
- NEVER scale services beyond available external instance capacity without adding instances

## Phase 3 — Rollback

1. If task definition was modified, revert to previous version
2. If service desired count was changed, restore previous value: `aws ecs update-service --cluster <cluster-name> --service <service-name> --desired-count <previous-count>`
3. If placement constraints were modified, restore previous configuration
4. Verify tasks are placed successfully after rollback
5. Document the placement failure and capacity requirements

## Output Format

```yaml
root_cause: "task_placement — <specific_cause>"
evidence:
  - type: placement_error
    content: "<service events and resource details>"
severity: HIGH
mitigation:
  immediate: "Fix task definition or add capacity"
  long_term: "Implement capacity monitoring and alerting"
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
