---
title: "E1 — Cost Estimation Issues"
description: "Diagnose cost estimation accuracy and configuration"
status: active
severity: LOW
triggers: ["cost estimation wrong", "unexpected costs", "billing question"]
owner: devops-agent
objective: "Resolve cost estimation issues"
context: "Cost estimation provides approximate monthly costs."
---

## Phase 1 — Triage

MUST:
- Get cost estimation: `aws devops-guru get-cost-estimation`
- Check resource count in coverage
- Verify estimation matches expectations

SHOULD:
- Compare with actual billing
- Check for resource count changes

## Phase 2 — Remediate

MUST:
- Adjust coverage to manage costs
- Remove unnecessary resources

## Common Issues

- symptoms: "Costs higher than estimated"
  diagnosis: "Resource count increased or high API call volume."
  resolution: "Review coverage. Remove unnecessary resources."

## Output Format

```yaml
root_cause: "cost_estimation — <specific_cause>"
evidence:
  - type: cost_data
    content: "<cost estimation>"
severity: LOW
mitigation:
  immediate: "Review coverage"
  long_term: "Monitor costs monthly"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Cost estimation is financial planning data |
| Blast Radius | LOW — Cost estimation is read-only |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | LOW — Cost estimation is operational |

## Escalation Conditions

- Escalate to finance if actual costs significantly exceed estimates
- Escalate to operations if cost reduction requires coverage changes

## Data Sensitivity

- **Cost estimates** are financial planning data — low sensitivity

## PROHIBITED Actions

- Do NOT present cost estimates as exact billing amounts
- Do NOT reduce coverage solely based on cost estimates without understanding monitoring impact

## Phase 3 — Rollback

- No rollback needed — cost estimation is read-only
- If coverage was reduced for cost reasons, restore if monitoring gaps are unacceptable

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
