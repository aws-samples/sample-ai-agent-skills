---
title: "B1 — Anomaly Detection Issues"
description: "Diagnose anomaly detection accuracy and coverage"
status: active
severity: MEDIUM
triggers: ["anomaly not detected", "missed anomaly", "detection gap"]
owner: devops-agent
objective: "Resolve anomaly detection issues"
context: "Anomaly detection uses ML models on CloudWatch metrics."
---

## Phase 1 — Triage

MUST:
- Check resource coverage
- Verify CloudWatch metrics are being published
- Check if resource has sufficient baseline data
- Review anomaly history

SHOULD:
- Check metric granularity
- Verify custom metrics are included

## Phase 2 — Remediate

MUST:
- Ensure resources are covered
- Verify metric collection
- Wait for baseline if new resource

## Common Issues

- symptoms: "Known issue not detected as anomaly"
  diagnosis: "Resource not covered or metric not monitored."
  resolution: "Add resource to coverage. Verify metric publication."

## Output Format

```yaml
root_cause: "anomaly_detection — <specific_cause>"
evidence:
  - type: coverage_status
    content: "<resource coverage>"
severity: MEDIUM
mitigation:
  immediate: "Fix coverage gaps"
  long_term: "Review detection coverage"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Anomaly data reveals operational behavior patterns |
| Blast Radius | LOW — Detection is read-only monitoring |
| Rollback Complexity | LOW — Coverage changes can be reverted |
| Compliance Scope | MEDIUM — Anomaly detection supports availability monitoring |

## Escalation Conditions

- Escalate to operations if known issues are not being detected
- Escalate to application owners if anomalies indicate application problems

## Data Sensitivity

- **Anomaly patterns** reveal resource behavior — operational data
- **CloudWatch metrics** used for detection are standard infrastructure data

## PROHIBITED Actions

- Do NOT remove resources from coverage without understanding the monitoring gap
- Do NOT ignore undetected anomalies — they indicate coverage gaps

## Phase 3 — Rollback

- Restore previous coverage configuration
- Verify CloudWatch metrics are being published for covered resources
- Wait for baseline to rebuild after coverage changes

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
