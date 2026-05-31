---
title: "D1 — ECS Service Recommendations"
description: "Diagnose ECS Fargate service recommendations"
status: active
severity: MEDIUM
triggers: ["ECS recommendation", "Fargate sizing", "task sizing"]
owner: devops-agent
objective: "Resolve ECS recommendation issues"
context: "ECS recommendations analyze task CPU and memory. Requires Container Insights."
---

## Phase 1 — Triage

MUST:
- Get recommendations: `aws compute-optimizer get-ecs-service-recommendations --service-arns <arn>`
- Verify Container Insights is enabled
- Check service has 14+ days of data

SHOULD:
- Verify Fargate launch type
- Check task definition configuration

## Phase 2 — Remediate

MUST:
- Enable Container Insights if not active
- Wait for data if new service

## Common Issues

- symptoms: "No ECS recommendations"
  diagnosis: "Container Insights not enabled or service too new."
  resolution: "Enable Container Insights. Wait 14+ days."

## Output Format

```yaml
root_cause: "ecs_recommendations — <specific_cause>"
evidence:
  - type: recommendation
    content: "<ECS recommendation>"
severity: MEDIUM
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Recommendations based on Container Insights metrics |
| Blast Radius | HIGH — Task sizing changes affect all running tasks in the service |
| Rollback Complexity | MEDIUM — Requires task definition update and service redeployment |
| Compliance Scope | LOW — Advisory recommendations only |

## Escalation Conditions

- Escalate to service owners before changing Fargate task sizing in production
- Escalate to operations if task sizing reduction causes OOM kills or CPU throttling

## Data Sensitivity

- **Service ARNs and task metrics** are standard operational data
- **Container Insights data** contains resource utilization, not application data

## PROHIBITED Actions

- Do NOT change task sizing on production services without testing in staging
- Do NOT reduce CPU/memory below application minimum requirements
- Do NOT update all services simultaneously — use staged rollout

## Phase 3 — Rollback

- Update task definition to previous CPU/memory values
- Force new deployment: `aws ecs update-service --cluster <cluster> --service <service> --force-new-deployment`
- Monitor service health and task stability after rollback

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
