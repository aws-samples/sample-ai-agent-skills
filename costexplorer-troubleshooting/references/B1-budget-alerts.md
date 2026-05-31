---
title: "B1 — Budget Alerts"
description: "Diagnose budget alert notification failures"
status: active
severity: HIGH
triggers:
  - "budget alert not firing"
  - "no budget notification"
  - "budget threshold exceeded no alert"
  - "SNS notification missing"
owner: devops-agent
objective: "Resolve budget alert configuration and notification delivery issues"
context: "Budget alerts evaluate periodically (not real-time). Notifications are sent via email or SNS. SNS topics must grant budgets.amazonaws.com publish permission. Budget evaluation has up to 24-hour delay matching Cost Explorer data."
---

## Phase 1 — Triage

MUST:
- Check budget configuration: `aws budgets describe-budgets --account-id <account-id>`
- Check notification settings: `aws budgets describe-notifications-for-budget --account-id <account-id> --budget-name <name>`
- Verify SNS topic policy allows budgets service: `aws sns get-topic-attributes --topic-arn <arn>`
- Check if threshold has actually been exceeded (considering data delay)

SHOULD:
- Verify subscriber endpoints: `aws budgets describe-subscribers-for-notification --account-id <account-id> --budget-name <name> --notification Type=ACTUAL,ComparisonOperator=GREATER_THAN,Threshold=<pct>`
- Check email delivery (spam folder)
- Verify budget time period matches expected evaluation window

MAY:
- Check CloudTrail for budget API events
- Review budget history for past notifications

## Phase 2 — Remediate

MUST:
- Add SNS topic policy for budgets.amazonaws.com
- Verify subscriber email addresses are confirmed
- Ensure budget threshold and comparison operator are correct

SHOULD:
- Set up multiple notification thresholds (50%, 80%, 100%)
- Use both email and SNS for redundancy
- Test with a low threshold to verify notification delivery

MAY:
- Set up budget actions for automated response
- Create CloudWatch alarms on EstimatedCharges as backup

## Common Issues

- symptoms: "Budget exceeded but no notification received"
  diagnosis: "SNS topic policy missing budgets.amazonaws.com permission, or data delay."
  resolution: "Add budgets service to SNS topic policy. Wait for budget evaluation cycle."

- symptoms: "Email notifications going to spam"
  diagnosis: "Budget notification emails filtered by email provider."
  resolution: "Whitelist AWS notification sender. Use SNS with Lambda for custom delivery."

- symptoms: "Budget shows $0 actual cost"
  diagnosis: "Budget created for wrong account, service, or time period."
  resolution: "Verify budget filters match the intended scope."

## Output Format

```yaml
root_cause: "budget_alerts — <specific_cause>"
evidence:
  - type: budget_config
    content: "<budget settings and thresholds>"
  - type: notification_config
    content: "<notification subscribers and SNS topic>"
severity: HIGH
mitigation:
  immediate: "Fix SNS permissions, subscriber config, or budget filters"
  long_term: "Implement multi-threshold alerts with redundant delivery"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Budget thresholds reveal financial targets |
| Blast Radius | MEDIUM — Missing alerts can lead to uncontrolled spending |
| Rollback Complexity | LOW — Budget and notification settings can be updated immediately |
| Compliance Scope | HIGH — Budget alerts are a financial governance control |

## Escalation Conditions

- Escalate IMMEDIATELY to finance if budget thresholds are exceeded without alerts firing
- Escalate to operations if SNS delivery failures prevent alert receipt
- Escalate to management if budget actions (SCPs, instance stops) fail to execute

## Data Sensitivity

- **Budget amounts and thresholds** are financial data — restrict access
- **SNS topic ARNs** and subscriber emails are operational data
- **Budget action configurations** may affect production resources

## PROHIBITED Actions

- Do NOT disable budget alerts without finance team approval
- Do NOT change budget thresholds without documenting the change and reason
- Do NOT configure budget actions that stop production instances without approval workflows
- Do NOT share budget details outside authorized finance and operations teams

## Phase 3 — Rollback

- Restore previous budget configuration: `aws budgets update-budget --account-id <id> --new-budget <previous-config>`
- Restore SNS topic policy if it was modified
- Verify alert delivery by testing with a low threshold
- Document any period where alerts were non-functional

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
