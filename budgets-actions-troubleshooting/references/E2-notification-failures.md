---
title: "E2 — Notification Failures"
description: "Diagnose budget notification delivery failures"
status: active
severity: HIGH
triggers: ["notification not received", "sns delivery failed", "email not received", "budget alert missing"]
owner: devops-agent
objective: "Resolve budget notification delivery failures"
context: "Budget notifications use SNS for delivery. Email subscribers must confirm subscriptions. Delivery failures can be caused by unconfirmed subscriptions, invalid endpoints, or SNS issues."
---

## Phase 1 — Triage

MUST:
- Check SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify subscription confirmation status (look for PendingConfirmation)
- Check SNS topic policy allows budgets.amazonaws.com to publish: `aws sns get-topic-attributes --topic-arn <topic-arn>`

SHOULD:
- Check CloudWatch metrics for SNS delivery: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic-name> --start-time 2025-04-01 --end-time 2025-04-12 --period 86400 --statistics Sum`
- Verify email addresses are correct and not bouncing

MAY:
- Check CloudWatch Logs for SNS delivery logs (if enabled)
- Verify corporate email filters are not blocking AWS notifications

## Phase 2 — Remediate

MUST:
- Confirm pending SNS subscriptions (resend confirmation if needed)
- Fix SNS topic policy to allow budgets.amazonaws.com: add `"Service": "budgets.amazonaws.com"` to the publish permission
- Verify subscriber endpoints are valid and reachable

SHOULD:
- Add multiple notification channels (email, SMS, Lambda, Chatbot)
- Enable SNS delivery logging for troubleshooting: `aws sns set-topic-attributes --topic-arn <arn> --attribute-name DeliveryPolicy --attribute-value file://delivery-policy.json`

MAY:
- Set up AWS Chatbot for Slack/Teams notifications
- Implement Lambda subscriber for custom notification handling

## Safety Ratings

```
safety_ratings:
  - "sns list-subscriptions-by-topic: GREEN — read-only listing"
  - "sns get-topic-attributes: GREEN — read-only topic check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "sns set-topic-attributes: YELLOW — modifies topic configuration"
```

## Escalation Conditions

- All notification channels failing simultaneously
- SNS topic deleted or permissions revoked
- Budget actions executing without notification to approvers
- Corporate email system blocking all AWS notifications

## Data Sensitivity

- SNS subscriptions: MEDIUM — contains email addresses and endpoints
- Topic policy: LOW — contains access policy
- Delivery metrics: LOW — contains delivery statistics
- Email addresses: MEDIUM — personally identifiable information

## Prohibited Actions

- NEVER add email subscribers without their consent
- NEVER disable SNS encryption on topics carrying budget data
- NEVER make SNS topics publicly accessible
- NEVER remove all subscribers from topics linked to budget actions

## Phase 3 — Rollback

1. Restore SNS topic policy if modified incorrectly: `aws sns set-topic-attributes --topic-arn <arn> --attribute-name Policy --attribute-value file://original-policy.json`
2. Remove incorrectly added subscribers: `aws sns unsubscribe --subscription-arn <sub-arn>`
3. Revert delivery policy changes if they caused issues
4. Verify notifications are working after changes
5. Document notification configuration changes

## Common Issues

- symptoms: "SNS subscription shows PendingConfirmation"
  diagnosis: "Subscriber has not clicked the confirmation link in the email."
  resolution: "Resend confirmation or have subscriber check spam folder. Use aws sns subscribe to create a new subscription."

## Output Format

```yaml
root_cause: "notification_failure — <specific_cause>"
evidence:
  - type: notification_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix notification delivery"
  long_term: "Set up multiple notification channels with monitoring"
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
