---
title: "J1 — Placement Constraint Issues"
description: "Diagnose task placement failures caused by placement constraints"
status: active
severity: MEDIUM
triggers:
  - "Cannot place task"
  - "Placement constraint"
  - "No matching instances"
owner: devops-agent
objective: "Fix placement constraints to allow task placement"
context: "Placement constraints restrict which container instances can run a task. Types: distinctInstance (one task per instance), memberOf (attribute expression). Overly restrictive constraints prevent placement."
---

## Phase 1 — Triage

MUST:
- Check task definition and service placement constraints
- List container instance attributes: `aws ecs describe-container-instances` → attributes
- Verify instances have the required attributes
- Check if distinctInstance constraint is preventing placement (need enough instances)

SHOULD:
- Check if custom attributes are set on instances
- Verify AZ spread constraints match available AZs
- Check if the constraint expression syntax is correct

MAY:
- Test without constraints to confirm they're the issue
- Check if placement strategies (binpack, spread) are conflicting with constraints

## Common Issues

- symptoms: "memberOf constraint for attribute not found on any instance"
  diagnosis: "Custom attribute not set on container instances."
  resolution: "Add the attribute to instances: `aws ecs put-attributes`."

- symptoms: "distinctInstance with more tasks than instances"
  diagnosis: "Can't place more tasks than instances with distinctInstance constraint."
  resolution: "Add more instances or remove the distinctInstance constraint."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Modify configuration: YELLOW - Changes service behavior, verify in non-production first"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
3. If capacity provider strategy was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --capacity-provider-strategy <original>`
## Output Format

```yaml
root_cause: "placement_constraint — <detail>"
evidence:
  - type: ecs_service
    content: "<constraint expression and instance attributes>"
severity: MEDIUM
mitigation:
  immediate: "Fix constraint or add required attributes to instances"
  long_term: "Document placement requirements, use Fargate to avoid placement complexity"
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
