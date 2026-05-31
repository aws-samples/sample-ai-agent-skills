---
title: "D2 — SNS Notification Errors"
description: "Diagnose SNS notification delivery failures for Device Defender alerts"
status: active
severity: HIGH
triggers: ["SNS delivery failed", "notification not received", "email alert missing", "Lambda not triggered by violation"]
owner: devops-agent
objective: "Resolve SNS notification delivery issues including topic configuration, subscriptions, and delivery failures"
context: "Device Defender sends violation notifications to SNS topics configured as alert targets. SNS then delivers to subscribed endpoints (email, Lambda, SQS, HTTP). Failures can occur at the IoT→SNS or SNS→endpoint stage."
---

## Phase 1 — Triage

MUST:
- Check SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify subscription confirmation status (must be Confirmed, not PendingConfirmation)
- Check SNS delivery metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Verify SNS topic policy allows iot.amazonaws.com to publish
- Check endpoint-specific delivery logs (Lambda invocation, SQS messages)
- Review SNS delivery status logging if enabled

MAY:
- Publish test message to SNS topic: `aws sns publish --topic-arn <topic-arn> --message "test"`
- Check email spam filters for SNS notification emails

## Phase 2 — Remediate

MUST:
- Confirm pending SNS subscriptions
- Fix SNS topic policy to allow IoT service principal
- Resolve endpoint delivery failures (Lambda errors, SQS permissions, email bounces)

SHOULD:
- Enable SNS delivery status logging for troubleshooting
- Configure dead-letter queue for failed deliveries

MAY:
- Set up CloudWatch alarms for SNS delivery failures
- Configure message filtering to reduce notification volume

## Safety Ratings

```
safety_ratings:
  - "sns list-subscriptions-by-topic: GREEN — read-only listing"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "sns publish (test): YELLOW — sends message to all subscribers"
  - "sns set-topic-attributes: YELLOW — modifies topic configuration"
```

## Escalation Conditions

- SNS delivery failures persist after fixing topic policy and subscriptions
- Critical violation notifications not delivered for extended period
- SNS topic throttled due to high violation volume
- Cross-account SNS delivery requires additional configuration

## Data Sensitivity

- `aws sns list-subscriptions-by-topic`: MEDIUM — reveals notification endpoints (emails, Lambda ARNs)
- `SNS delivery metrics`: LOW — returns delivery statistics
- `violation notification content`: MEDIUM — contains device identifiers and violation details
- `SNS topic policy`: MEDIUM — reveals access control configuration

## Prohibited Actions

- NEVER subscribe unauthorized endpoints to violation notification topics
- NEVER disable SNS encryption for topics carrying violation data
- NEVER publish test messages that could trigger automated remediation workflows
- NEVER share SNS notification content containing device details publicly

## Phase 3 — Rollback

1. If SNS topic was modified incorrectly, restore previous policy and configuration
2. If subscriptions were changed, restore previous subscription list
3. If delivery status logging was enabled, verify it's not causing excessive CloudWatch costs
4. Verify notification delivery is restored by checking recent violation alerts
5. Document the notification issue and resolution

## Common Issues

- symptoms: "SNS subscription in PendingConfirmation — email not confirmed"
  diagnosis: "Subscription confirmation email was not clicked or went to spam."
  resolution: "Re-subscribe and confirm. Check spam folder for confirmation email."

## Output Format

```yaml
root_cause: "sns_notification — <specific_cause>"
evidence:
  - type: sns_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix notification delivery"
  long_term: "Enable delivery status logging and dead-letter queue"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
