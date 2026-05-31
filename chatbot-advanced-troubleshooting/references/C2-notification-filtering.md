---
title: "C2 — Notification Filtering"
description: "Diagnose notification filtering issues including missing notifications, excessive notifications, and event pattern configuration"
status: active
severity: MEDIUM
triggers: ["notifications missing", "too many notifications", "notification filter not working", "event not delivered"]
owner: devops-agent
objective: "Resolve notification filtering issues to ensure correct event delivery"
context: "Notification filtering controls which events are delivered to chat channels. Filters use notification preferences and event patterns. Non-matching events are silently dropped with no error."
---

## Phase 1 — Triage

MUST:
- Check channel notification preferences: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>`
- Verify SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Check if events are being published to the SNS topic

SHOULD:
- Review notification preference settings (event types, severity levels)
- Check if event patterns are configured and match expected events
- Verify SNS subscription filter policies

MAY:
- Publish a test event to verify delivery: `aws sns publish --topic-arn <arn> --message '{"test":"notification"}'`
- Check CloudWatch metrics for SNS delivery

## Phase 2 — Remediate

MUST:
- Update notification preferences to include desired event types
- Fix event pattern matching rules
- Ensure SNS subscriptions are active and confirmed

SHOULD:
- Configure severity-based filtering to reduce noise
- Set up separate channels for different notification types
- Test filter changes with sample events

MAY:
- Implement EventBridge rules for advanced event filtering before SNS
- Create notification routing based on event attributes

## Safety Ratings

```
safety_ratings:
  - "describe-slack-channel-configurations: GREEN — read-only configuration"
  - "sns list-subscriptions-by-topic: GREEN — read-only subscription listing"
  - "update-slack-channel-configuration: YELLOW — modifies notification settings"
  - "sns publish: YELLOW — sends message to topic subscribers"
```

## Escalation Conditions

- Critical alerts not being delivered to the operations channel
- Notification flood overwhelming the channel and hiding important alerts
- Filter configuration too complex to manage effectively
- Cross-account notifications not being filtered correctly

## Data Sensitivity

- `aws chatbot describe-slack-channel-configurations`: LOW — configuration metadata
- `aws sns list-subscriptions-by-topic`: LOW — subscription endpoints
- Notification content: MEDIUM — contains AWS event details
- Event patterns: LOW — contains filter rules

## Prohibited Actions

- NEVER disable all notification filtering to troubleshoot — it floods the channel
- NEVER modify SNS topic subscriptions without understanding all consumers
- NEVER remove severity filtering for production alert channels
- NEVER test notification delivery with production SNS topics during business hours

## Phase 3 — Rollback

1. If notification preferences were changed, restore previous settings
2. If SNS subscriptions were modified, restore previous state
3. If event patterns were changed, revert to previous patterns
4. Verify notification delivery resumes correctly after rollback
5. Monitor the channel for correct notification behavior

## Common Issues

- symptoms: "No notifications appearing in the channel"
  diagnosis: "SNS subscription is not confirmed, notification preferences filter out all events, or no events are published."
  resolution: "Check subscription status. Review notification preferences. Verify events are published to the topic."

- symptoms: "Channel flooded with too many notifications"
  diagnosis: "Notification preferences are too broad or no severity filtering is configured."
  resolution: "Configure notification preferences to filter by event type and severity. Use separate channels for different alert levels."

- symptoms: "Some event types delivered but others are not"
  diagnosis: "Notification preferences or event patterns exclude certain event types."
  resolution: "Review notification preferences for the missing event types. Update event patterns to match."

## Output Format

```yaml
root_cause: "notification_filtering — <specific_cause>"
evidence:
  - type: filter_error
    content: "<notification preference and delivery details>"
severity: MEDIUM
mitigation:
  immediate: "Fix notification filtering"
  long_term: "Implement structured notification routing"
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
