---
title: "A1 — Reactive Insight Issues"
description: "Diagnose missing or delayed reactive insights"
status: active
severity: HIGH
triggers: ["no insights", "missing insights", "insight delayed"]
owner: devops-agent
objective: "Resolve reactive insight issues"
context: "Reactive insights detect ongoing operational issues from anomalies."
---

## Phase 1 — Triage

MUST:
- Check account health: `aws devops-guru describe-account-health`
- List insights: `aws devops-guru list-insights --status-filter ONGOING`
- Verify resource coverage is configured
- Check if learning period has completed

SHOULD:
- Verify CloudWatch metrics are being collected
- Check for service health issues

## Phase 2 — Remediate

MUST:
- Configure resource coverage if missing
- Wait for learning period to complete
- Verify CloudWatch metric collection

## Common Issues

- symptoms: "No insights after enabling DevOps Guru"
  diagnosis: "Learning period not complete or no coverage configured."
  resolution: "Configure coverage. Wait hours-days for baseline learning."

## Output Format

```yaml
root_cause: "reactive_insights — <specific_cause>"
evidence:
  - type: account_health
    content: "<account health status>"
severity: HIGH
mitigation:
  immediate: "Fix coverage and wait for learning"
  long_term: "Monitor insight generation"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Insights reveal operational anomalies in monitored resources |
| Blast Radius | LOW — Insights are read-only analysis |
| Rollback Complexity | LOW — Coverage changes can be reverted |
| Compliance Scope | MEDIUM — Operational insights support availability requirements |

## Escalation Conditions

- Escalate to operations if reactive insights indicate ongoing service degradation
- Escalate to application owners if insights reveal application-level issues
- Escalate to AWS Support if insights are not generated despite anomalous behavior

## Data Sensitivity

- **Insight details** reveal operational anomalies — restrict to operations team
- **Resource identifiers** in insights are standard infrastructure data

## PROHIBITED Actions

- Do NOT ignore reactive insights indicating ongoing issues
- Do NOT close insights without investigating the root cause
- Do NOT modify resource coverage during active incident investigation

## Phase 3 — Rollback

- Restore previous coverage configuration if it was changed
- Reopen closed insights if they were prematurely dismissed
- Wait for learning period to complete if coverage was recently changed

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
