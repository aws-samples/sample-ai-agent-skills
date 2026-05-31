---
title: "E2 — Cost Optimization"
description: "Optimize DevOps Guru costs"
status: active
severity: LOW
triggers: ["reduce costs", "cost optimization", "too expensive"]
owner: devops-agent
objective: "Optimize DevOps Guru costs"
context: "Costs scale with monitored resources."
---

## Phase 1 — Triage

MUST:
- Review current coverage scope
- Check cost estimation
- Identify low-value monitored resources

## Phase 2 — Remediate

MUST:
- Remove non-critical resources from coverage
- Use tag-based coverage for targeted monitoring

## Common Issues

- symptoms: "DevOps Guru costs too high"
  diagnosis: "Too many resources in coverage."
  resolution: "Narrow coverage to critical resources. Use tags for precision."

## Output Format

```yaml
root_cause: "cost_optimization — <specific_cause>"
evidence:
  - type: cost_analysis
    content: "<cost breakdown>"
severity: LOW
mitigation:
  immediate: "Reduce coverage"
  long_term: "Regular cost reviews"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Cost optimization is operational planning |
| Blast Radius | MEDIUM — Reducing coverage creates monitoring blind spots |
| Rollback Complexity | LOW — Coverage can be restored |
| Compliance Scope | MEDIUM — Coverage reduction may affect availability monitoring requirements |

## Escalation Conditions

- Escalate to operations before removing resources from coverage
- Escalate to management if cost vs monitoring tradeoffs need decision

## Data Sensitivity

- **Coverage decisions** are operational planning data

## PROHIBITED Actions

- Do NOT remove critical production resources from coverage for cost savings
- Do NOT optimize costs without documenting the monitoring impact

## Phase 3 — Rollback

- Restore previous coverage scope
- Re-add resources that were removed for cost optimization
- Verify monitoring is restored for all critical resources

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
