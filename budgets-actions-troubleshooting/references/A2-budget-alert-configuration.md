---
title: "A2 — Budget Alert Configuration"
description: "Diagnose budget alert and notification configuration issues"
status: active
severity: MEDIUM
triggers: ["budget alert not working", "notification not received", "alert configuration", "budget threshold"]
owner: devops-agent
objective: "Resolve budget alert configuration issues"
context: "Budget alerts notify subscribers when spending approaches or exceeds thresholds. Alerts use SNS for delivery and require confirmed subscriptions."
---

## Phase 1 — Triage

MUST:
- Check notification configuration: `aws budgets describe-notifications-for-budget --account-id <account-id> --budget-name <name>`
- List subscribers: `aws budgets describe-subscribers-for-notification --account-id <account-id> --budget-name <name> --notification '{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80}'`
- Verify SNS topic and subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`

SHOULD:
- Check if SNS subscriptions are confirmed (PendingConfirmation status)
- Verify budget threshold values and comparison operators

MAY:
- Check SNS delivery logs in CloudWatch
- Verify email addresses are correct

## Phase 2 — Remediate

MUST:
- Confirm SNS subscriptions (subscribers must click confirmation link)
- Fix notification thresholds and comparison operators
- Create notifications if missing: `aws budgets create-notification --account-id <account-id> --budget-name <name> --notification '{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80}' --subscribers '[{"SubscriptionType":"SNS","Address":"<topic-arn>"}]'`

SHOULD:
- Set up multiple threshold levels (50%, 80%, 100%)
- Use both ACTUAL and FORECASTED notification types

MAY:
- Add Lambda subscribers for automated responses
- Configure AWS Chatbot for Slack/Teams notifications

## Safety Ratings

```
safety_ratings:
  - "budgets describe-notifications-for-budget: GREEN — read-only listing"
  - "budgets describe-subscribers-for-notification: GREEN — read-only listing"
  - "sns list-subscriptions-by-topic: GREEN — read-only listing"
  - "budgets create-notification: YELLOW — adds notification trigger"
```

## Escalation Conditions

- SNS delivery failures persist after subscription confirmation
- Budget notifications not triggering despite threshold breach
- Email notifications blocked by corporate email filters
- Multiple notification channels all failing simultaneously

## Data Sensitivity

- Notification configuration: LOW — contains threshold values
- Subscriber addresses: MEDIUM — contains email addresses or SNS ARNs
- SNS topic configuration: LOW — contains topic metadata
- Budget thresholds: MEDIUM — reveals cost governance strategy

## Prohibited Actions

- NEVER remove all notifications from a budget with active actions
- NEVER add subscribers without their consent (email addresses)
- NEVER set thresholds above 100% as the only alert (too late for action)
- NEVER disable notifications during active budget monitoring periods

## Phase 3 — Rollback

1. Delete incorrectly created notifications: `aws budgets delete-notification --account-id <account-id> --budget-name <name> --notification <notification-json>`
2. Remove incorrect subscribers from notifications
3. Restore original threshold values if changed
4. Verify notifications are working after changes
5. Document notification configuration changes

## Common Issues

- symptoms: "Budget threshold exceeded but no notification received"
  diagnosis: "SNS subscription not confirmed or email in spam folder."
  resolution: "Check SNS subscription status. Resend confirmation. Check spam/junk folders."

## Output Format

```yaml
root_cause: "alert_configuration — <specific_cause>"
evidence:
  - type: alert_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix alert configuration"
  long_term: "Set up multiple notification channels"
```

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
