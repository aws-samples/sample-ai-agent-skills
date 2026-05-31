---
title: "B2 — ECS on Outposts Issues"
description: "Diagnose ECS task and service problems running on AWS Outposts"
status: active
severity: HIGH
triggers:
  - "ECS on Outposts"
  - "ECS task outpost"
  - "container outpost"
  - "ECS capacity provider outpost"
owner: devops-agent
objective: "Resolve ECS task placement, execution, and networking issues on Outposts"
context: "ECS on Outposts runs containers on EC2 instances deployed on the Outpost. Tasks require service link connectivity for placement. ECS capacity providers can manage Outpost EC2 instances. Fargate is NOT supported on Outposts."
---

## Phase 1 — Triage

MUST:
- List ECS clusters: `aws ecs list-clusters`
- Describe cluster: `aws ecs describe-clusters --clusters <cluster-name> --include STATISTICS`
- List services: `aws ecs list-services --cluster <cluster-name>`
- Describe service events: `aws ecs describe-services --cluster <cluster-name> --services <service-name> --query 'services[*].events[:5]'`
- Check container instances on Outpost: `aws ecs list-container-instances --cluster <cluster-name> --filter "attribute:ecs.outpost-arn == <outpost-arn>"`

SHOULD:
- Check task definition compatibility: `aws ecs describe-task-definition --task-definition <task-def>`
- Verify container instance status: `aws ecs describe-container-instances --cluster <cluster-name> --container-instances <instance-arn>`
- Check Outpost connectivity status

MAY:
- Review capacity provider configuration
- Check CloudWatch Container Insights if enabled

## Phase 2 — Remediate

MUST:
- If task placement fails: verify container instances have sufficient resources on the Outpost
- If Fargate requested: switch to EC2 launch type — Fargate is not supported on Outposts
- If service link down: existing tasks continue but new placements fail

SHOULD:
- Use placement constraints to target Outpost instances: `attribute:ecs.outpost-arn == <outpost-arn>`
- Ensure ECS agent is running on Outpost EC2 instances
- Verify task networking mode is compatible (awsvpc, bridge, host)

MAY:
- Implement ECS capacity providers for Outpost auto-scaling groups
- Set up service discovery for Outpost-based services

## Common Issues

- symptoms: "ECS service unable to place tasks"
  diagnosis: "No container instances with sufficient resources on the Outpost."
  resolution: "Scale Outpost EC2 instances or reduce task resource requirements."

- symptoms: "Task starts but fails health checks"
  diagnosis: "Network connectivity between Outpost and load balancer or service discovery."
  resolution: "Verify route tables, security groups, and target group configuration."

## Output Format

```yaml
root_cause: "ecs_outpost — <specific_cause>"
evidence:
  - type: service_events
    content: "<ECS service events>"
  - type: container_instances
    content: "<container instance status>"
severity: HIGH
mitigation:
  immediate: "Resolve task placement or execution issue"
  long_term: "Implement capacity providers and placement strategies"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — ECS management is operational
- Service Disruption Risk: MEDIUM — ECS issues affect containerized workloads
- Compliance Impact: MEDIUM — container workloads must meet Outpost compliance requirements

## Escalation Conditions
- ECS tasks fail to launch on Outpost instances
- Container networking issues specific to Outpost
- ECS service discovery fails on Outpost
- Task placement constraints cannot be satisfied
- ECS agent connectivity issues on Outpost instances

## Data Sensitivity
- Container configurations — INTERNAL
- Task definitions — INTERNAL
- Container logs — INTERNAL (may contain CONFIDENTIAL data)
- Service discovery configurations — INTERNAL
- Never expose container secrets or environment variables

## Prohibited Actions
- Do NOT deploy untested container images to Outpost ECS clusters
- Do NOT bypass task placement constraints
- Do NOT expose container ports to untrusted networks
- Do NOT store secrets in container environment variables
- Do NOT ignore ECS agent health warnings
- Do NOT run privileged containers without security review

## Phase 3 — Rollback
- Roll back ECS service to previous task definition
- Restore previous container configurations
- Revert networking changes that affected ECS connectivity
- Re-register container instances if deregistered
- Verify ECS service health after rollback
- Document ECS issues and container deployment changes

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
