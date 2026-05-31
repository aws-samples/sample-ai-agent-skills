---
title: "D2 — Notification Filtering Issues"
description: "Diagnose notification filtering and severity routing"
status: active
severity: LOW
triggers: ["too many notifications", "filter not working", "severity routing"]
owner: devops-agent
objective: "Resolve notification filtering issues"
context: "Notifications can be filtered by severity and insight type."
---

## Phase 1 — Triage

MUST:
- Check notification channel configuration
- Review filter settings
- Verify severity levels

## Phase 2 — Remediate

MUST:
- Configure appropriate filters
- Set severity thresholds

## Common Issues

- symptoms: "Overwhelmed by low-severity notifications"
  diagnosis: "No severity filtering configured."
  resolution: "Configure notification filters for HIGH severity only."

## Output Format

```yaml
root_cause: "notification_filtering — <specific_cause>"
evidence:
  - type: filter_config
    content: "<filter settings>"
severity: LOW
mitigation:
  immediate: "Configure filters"
  long_term: "Tune notification strategy"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Filter configuration is operational metadata |
| Blast Radius | LOW — Filter changes affect notification delivery only |
| Rollback Complexity | LOW — Filters can be reconfigured |
| Compliance Scope | LOW — Notification filtering is operational |

## Escalation Conditions

- Escalate to operations if filtering suppresses critical notifications

## Data Sensitivity

- **Filter settings** are operational configuration
- No sensitive data involved

## PROHIBITED Actions

- Do NOT filter out HIGH severity notifications without operations team approval
- Do NOT set filters so restrictive that critical insights are missed

## Phase 3 — Rollback

- Restore previous filter configuration
- Verify critical notifications are being delivered after changes

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
