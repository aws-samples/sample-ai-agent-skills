---
title: "F1 — Proactive Insight Configuration"
description: "Diagnose proactive insight setup and configuration"
status: active
severity: MEDIUM
triggers: ["proactive not working", "prediction config", "proactive setup"]
owner: devops-agent
objective: "Resolve proactive insight configuration"
context: "Proactive insights predict issues before they occur."
---

## Phase 1 — Triage

MUST:
- Verify proactive insights are enabled
- Check resource coverage for proactive monitoring
- Verify sufficient historical data

## Phase 2 — Remediate

MUST:
- Enable proactive insights
- Ensure adequate data history

## Common Issues

- symptoms: "Proactive insights not generating"
  diagnosis: "Feature not enabled or insufficient data."
  resolution: "Enable proactive insights. Wait for data accumulation."

## Output Format

```yaml
root_cause: "proactive_config — <specific_cause>"
evidence:
  - type: proactive_status
    content: "<proactive configuration>"
severity: MEDIUM
mitigation:
  immediate: "Enable and configure"
  long_term: "Monitor prediction quality"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Proactive configuration is operational |
| Blast Radius | LOW — Proactive insights are advisory |
| Rollback Complexity | LOW — Configuration can be changed |
| Compliance Scope | LOW — Proactive insights are operational tools |

## Escalation Conditions

- Escalate to operations if proactive insights are needed but not generating

## Data Sensitivity

- **Proactive configuration** is operational metadata

## PROHIBITED Actions

- Do NOT disable proactive insights without understanding the impact on predictive monitoring

## Phase 3 — Rollback

- Re-enable proactive insights if they were disabled
- Wait for sufficient data accumulation for predictions to resume

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
