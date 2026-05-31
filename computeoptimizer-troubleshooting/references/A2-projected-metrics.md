---
title: "A2 — EC2 Projected Metrics Issues"
description: "Diagnose projected metric accuracy and availability"
status: active
severity: LOW
triggers: ["projected metrics wrong", "metric projection", "performance prediction"]
owner: devops-agent
objective: "Resolve projected metrics issues"
context: "Projected metrics show expected performance with recommended instance types."
---

## Phase 1 — Triage

MUST:
- Get projected metrics: `aws compute-optimizer get-ec2-recommendation-projected-metrics --instance-arn <arn> --stat Maximum --period 86400 --start-time <start> --end-time <end>`
- Verify recommendation exists
- Check metric availability

## Phase 2 — Remediate

MUST:
- Ensure sufficient historical data
- Verify instance has recommendations

## Common Issues

- symptoms: "Projected metrics not available"
  diagnosis: "Insufficient data or no recommendation."
  resolution: "Wait for data. Verify recommendation exists."

## Output Format

```yaml
root_cause: "projected_metrics — <specific_cause>"
evidence:
  - type: metrics
    content: "<projected metric data>"
severity: LOW
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Projected metrics are derived from historical CloudWatch data |
| Blast Radius | LOW — Read-only projections; no infrastructure changes |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | LOW — Advisory data only |

## Escalation Conditions

- Escalate to operations if projected metrics show significant performance risk for recommended changes
- Escalate to AWS Support if projected metrics are unavailable for instances with recommendations

## Data Sensitivity

- **Projected metrics** are modeled estimates — low sensitivity
- **Instance identifiers** in projections are standard resource metadata

## PROHIBITED Actions

- Do NOT treat projected metrics as guaranteed performance outcomes
- Do NOT make capacity decisions based solely on projected metrics without real-world testing

## Phase 3 — Rollback

- No rollback needed — projected metrics are read-only advisory data
- If decisions were made based on inaccurate projections, revert the infrastructure changes per the relevant runbook

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
