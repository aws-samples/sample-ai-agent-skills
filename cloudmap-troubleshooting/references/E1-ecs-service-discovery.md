---
title: "E1 — Cloud Map ECS Service Discovery Issues"
description: "Diagnose ECS service discovery integration problems"
status: active
severity: HIGH
triggers:
  - "ECS service discovery"
  - "ECS not registering"
  - "ECS Cloud Map"
  - "task not discoverable"
owner: devops-agent
objective: "Identify and resolve ECS service discovery integration issues with Cloud Map"
context: "ECS service discovery automatically registers/deregisters task IPs with Cloud Map. ECS manages the Cloud Map service and instances. Issues stem from configuration mismatches, IAM permissions, or networking."
---

## Phase 1 — Triage

MUST:
- Check ECS service configuration: `aws ecs describe-services --cluster <cluster> --services <service> --query 'services[0].serviceRegistries'`
- Check Cloud Map service: `aws servicediscovery get-service --id <service-id>`
- List registered instances: `aws servicediscovery list-instances --service-id <service-id>`
- Check ECS task status: `aws ecs list-tasks --cluster <cluster> --service-name <service>`
- Verify tasks have IP addresses: `aws ecs describe-tasks --cluster <cluster> --tasks <task-arn> --query 'tasks[0].containers[0].networkInterfaces'`

SHOULD:
- Verify ECS service uses awsvpc network mode (required for service discovery)
- Check Cloud Map namespace matches ECS service configuration
- Verify DNS resolution from within the VPC

MAY:
- Check ECS service events for registration errors: `aws ecs describe-services --cluster <cluster> --services <service> --query 'services[0].events[:10]'`
- Verify security groups allow traffic on the service port

## Phase 2 — Remediate

MUST:
- Use awsvpc network mode for ECS tasks (required for service discovery)
- Ensure ECS service has correct serviceRegistries configuration
- Grant ECS task execution role Cloud Map permissions

SHOULD:
- Verify Cloud Map namespace and service exist before creating ECS service
- Use private DNS namespace for VPC-internal discovery
- Do NOT manually modify ECS-managed Cloud Map instances

MAY:
- Set up health checks on the Cloud Map service
- Monitor ECS service discovery registration events

## Common Issues

- symptoms: "ECS tasks running but not registered in Cloud Map"
  diagnosis: "ECS service not configured with serviceRegistries or wrong network mode."
  resolution: "Configure serviceRegistries on the ECS service and use awsvpc network mode."

- symptoms: "Instances registered but with wrong IP"
  diagnosis: "Task using bridge network mode instead of awsvpc."
  resolution: "Switch to awsvpc network mode for correct IP registration."

- symptoms: "Service discovery works but DNS returns unhealthy instances"
  diagnosis: "Health checks not configured or failing on healthy tasks."
  resolution: "Configure appropriate health checks on the Cloud Map service."

## Output Format

```yaml
root_cause: "ecs_service_discovery_issue — <specific_cause>"
evidence:
  - type: ecs_config
    content: "<service registries and network mode>"
  - type: cloudmap_instances
    content: "<registered instances>"
  - type: task_status
    content: "<ECS task details>"
severity: HIGH
mitigation:
  immediate: "Fix ECS service configuration or network mode"
  long_term: "Standardize ECS service discovery configuration in IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use awsvpc network mode for ECS tasks | YELLOW |
| Ensure ECS service has correct serviceRegistries | YELLOW |
| Grant ECS task execution role Cloud Map permissions | YELLOW |
| Verify Cloud Map namespace and service exist | GREEN |
| Use private DNS namespace for VPC-internal discovery | GREEN |
| Do NOT manually modify ECS-managed Cloud Map instances | GREEN |

## Escalation Conditions

- Fix requires modifying production ECS service configurations
- Fix involves changing network mode which requires service recreation

## Data Sensitivity

- **MEDIUM**: ECS service discovery configurations expose container networking and service topology
- Task IP addresses and port mappings reveal infrastructure details

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If ECS service was modified, restore previous service configuration (may require service recreation)
2. If network mode was changed, revert to previous mode (requires new task definition and service update)
3. If Cloud Map permissions were modified, revert IAM policy changes
4. Verify rollback by confirming ECS tasks are correctly registered in Cloud Map

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
