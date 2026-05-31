---
title: "B2 — False Positive Anomalies"
description: "Diagnose and reduce false positive anomaly detections"
status: active
severity: LOW
triggers: ["false positive", "wrong anomaly", "noise"]
owner: devops-agent
objective: "Reduce false positive anomalies"
context: "ML models may flag normal behavior changes as anomalies."
---

## Phase 1 — Triage

MUST:
- Review anomaly details
- Check if behavior change was expected (deployment, scaling)
- Verify baseline period was sufficient

SHOULD:
- Check for recent deployments or config changes
- Review anomaly patterns

## Phase 2 — Remediate

MUST:
- Close false positive insights with feedback
- Allow model to learn new patterns

SHOULD:
- Correlate anomalies with deployment events
- Provide feedback to improve model

## Common Issues

- symptoms: "Anomaly after every deployment"
  diagnosis: "Model flags deployment-related metric changes."
  resolution: "Close insights. Model learns deployment patterns over time."

## Output Format

```yaml
root_cause: "false_positive — <specific_cause>"
evidence:
  - type: anomaly_details
    content: "<anomaly analysis>"
severity: LOW
mitigation:
  immediate: "Close false positives"
  long_term: "Model improves with feedback"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — False positive management is operational |
| Blast Radius | LOW — Feedback affects model learning only |
| Rollback Complexity | LOW — Model adjusts over time |
| Compliance Scope | LOW — False positive tuning is operational |

## Escalation Conditions

- Escalate to operations if false positives overwhelm the team and real issues are missed

## Data Sensitivity

- **Insight feedback** is operational metadata
- No sensitive data involved in false positive management

## PROHIBITED Actions

- Do NOT close real anomalies as false positives without investigation
- Do NOT remove resources from coverage to eliminate false positives

## Phase 3 — Rollback

- No rollback needed — model learns from feedback over time
- If real anomalies were incorrectly closed, reopen and investigate

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
