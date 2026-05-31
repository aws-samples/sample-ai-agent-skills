---
title: "D2 — ECS Task Sizing Issues"
description: "Diagnose ECS task CPU and memory sizing"
status: active
severity: LOW
triggers: ["task over-provisioned", "task under-provisioned", "Fargate cost"]
owner: devops-agent
objective: "Resolve ECS task sizing issues"
context: "Task sizing affects cost and performance."
---

## Phase 1 — Triage

MUST:
- Review task CPU and memory recommendations
- Check current vs recommended configuration
- Verify workload patterns

## Phase 2 — Remediate

MUST:
- Update task definition with recommended sizing
- Test in staging before production

## Common Issues

- symptoms: "Tasks over-provisioned with 4 vCPU but using <1"
  diagnosis: "Task definition over-sized for workload."
  resolution: "Reduce to recommended CPU/memory. Test performance."

## Output Format

```yaml
root_cause: "task_sizing — <specific_cause>"
evidence:
  - type: task_analysis
    content: "<task sizing details>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Task sizing analysis uses resource utilization metrics |
| Blast Radius | MEDIUM — Incorrect sizing causes performance issues or wasted spend |
| Rollback Complexity | LOW — Task definition can be reverted quickly |
| Compliance Scope | LOW — No compliance impact |

## Escalation Conditions

- Escalate to service owners if sizing changes cause task failures
- Escalate to operations if under-sized tasks cause service degradation

## Data Sensitivity

- **Task utilization data** is standard operational metrics
- No application data is exposed through task sizing analysis

## PROHIBITED Actions

- Do NOT reduce task sizing below tested minimums for the application
- Do NOT change sizing during traffic peaks

## Phase 3 — Rollback

- Revert task definition to previous CPU/memory configuration
- Deploy the reverted task definition to the service
- Verify all tasks are healthy after rollback

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
