---
title: "D1 — SNS Notification Channel Issues"
description: "Diagnose notification channel configuration and delivery"
status: active
severity: MEDIUM
triggers: ["no notifications", "SNS not working", "notification missing"]
owner: devops-agent
objective: "Resolve notification channel issues"
context: "SNS topics must be configured as notification channels."
---

## Phase 1 — Triage

MUST:
- List channels: `aws devops-guru list-notification-channels`
- Verify SNS topic exists and has subscribers
- Check IAM permissions for DevOps Guru to publish to SNS
- Verify topic policy allows devops-guru.amazonaws.com

SHOULD:
- Test SNS topic independently
- Check for subscription confirmation

## Phase 2 — Remediate

MUST:
- Create notification channel if missing
- Fix SNS topic policy
- Confirm subscriptions

## Common Issues

- symptoms: "No notifications for new insights"
  diagnosis: "No notification channel configured."
  resolution: "Create SNS topic. Add as notification channel. Subscribe endpoints."

## Output Format

```yaml
root_cause: "sns_channels — <specific_cause>"
evidence:
  - type: channel_config
    content: "<notification configuration>"
severity: MEDIUM
mitigation:
  immediate: "Configure notifications"
  long_term: "Monitor notification delivery"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Notification channels deliver operational alerts |
| Blast Radius | MEDIUM — Missing notifications can delay incident response |
| Rollback Complexity | LOW — Channels can be reconfigured |
| Compliance Scope | MEDIUM — Timely notifications support incident response requirements |

## Escalation Conditions

- Escalate to operations if notifications are not being delivered for active insights
- Escalate to SNS administrators if topic policy issues prevent delivery

## Data Sensitivity

- **SNS topic ARNs** and subscriber endpoints are operational data
- **Notification content** contains insight details — restrict access

## PROHIBITED Actions

- Do NOT remove notification channels without setting up alternatives
- Do NOT modify SNS topic policies without verifying DevOps Guru publish access

## Phase 3 — Rollback

- Restore SNS topic policy to allow devops-guru.amazonaws.com publish access
- Recreate notification channel if it was deleted
- Verify notification delivery with a test

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
