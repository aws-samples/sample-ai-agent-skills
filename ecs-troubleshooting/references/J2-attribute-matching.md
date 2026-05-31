---
title: "J2 — Attribute Matching Issues"
description: "Diagnose task placement failures caused by instance attribute mismatches"
status: active
severity: MEDIUM
triggers:
  - "No matching container instances"
  - "Attribute not found"
  - "Instance type mismatch"
owner: devops-agent
objective: "Fix instance attribute configuration for task placement"
context: "ECS uses built-in attributes (instance type, AZ, AMI, OS) and custom attributes for placement. Tasks can require specific attributes via placement constraints. Missing attributes prevent placement."
---

## Phase 1 — Triage

MUST:
- Check placement constraints for attribute requirements
- List instance attributes: `aws ecs describe-container-instances` → attributes
- Verify built-in attributes: ecs.instance-type, ecs.availability-zone, ecs.ami-id, ecs.os-type
- Check for custom attribute requirements

SHOULD:
- Verify the attribute value matches exactly (case-sensitive)
- Check if the attribute was recently removed or changed
- Verify GPU attributes if GPU tasks are failing (ecs.gpu-count)

MAY:
- List all attributes across all instances to find matches
- Check if the ASG launch template creates instances with correct attributes

## Common Issues

- symptoms: "GPU task can't be placed, no instances with ecs.gpu-count"
  diagnosis: "No GPU instances in the cluster, or GPU instances not registered."
  resolution: "Add GPU instances to the cluster (p3, g4, etc.)."

- symptoms: "Custom attribute 'environment=production' not found"
  diagnosis: "Custom attribute not set on any container instance."
  resolution: "Set the attribute: `aws ecs put-attributes --cluster <cluster> --attributes name=environment,value=production,targetId=<instance-arn>`."


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
root_cause: "attribute_matching — <detail>"
evidence:
  - type: container_instance
    content: "<required vs available attributes>"
severity: MEDIUM
mitigation:
  immediate: "Add required attributes to instances"
  long_term: "Automate attribute assignment in instance bootstrap, document requirements"
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
