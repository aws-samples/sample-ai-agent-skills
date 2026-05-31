---
title: "B2 — Notification Setup"
description: "Diagnose health event notification delivery issues"
status: active
severity: MEDIUM
triggers:
  - "health notification not received"
  - "no health alerts"
  - "notification setup"
  - "health email notification"
owner: devops-agent
objective: "Resolve health event notification delivery issues"
context: "Health notifications are delivered via EventBridge rules targeting SNS, Lambda, SQS, or other services. There is no built-in email notification — it must be configured via EventBridge + SNS. Chatbot integration is available for Slack/Teams."
---

## Phase 1 — Triage

MUST:
- Check EventBridge rules for health events: `aws events list-rules --region us-east-1`
- Verify SNS topic subscription: `aws sns list-subscriptions-by-topic --topic-arn <arn>`
- Check SNS topic policy allows EventBridge: `aws sns get-topic-attributes --topic-arn <arn>`
- Verify email subscription is confirmed (pending subscriptions don't receive messages)

SHOULD:
- Check if notifications are going to spam folder
- Verify EventBridge rule is targeting the correct SNS topic
- Check CloudWatch metrics for SNS delivery failures

MAY:
- Check AWS Chatbot configuration for Slack/Teams notifications
- Review notification history in SNS delivery logs

## Phase 2 — Remediate

MUST:
- Create EventBridge rule targeting SNS topic for health events
- Add SNS topic policy allowing events.amazonaws.com to publish
- Confirm email subscriptions in SNS

SHOULD:
- Set up multiple notification channels (email, Slack, PagerDuty)
- Filter notifications by severity (issues only, or all categories)
- Test notification pipeline with a test event

MAY:
- Implement custom notification formatting with Lambda
- Set up escalation paths based on event severity

## Common Issues

- symptoms: "No email notifications for health events"
  diagnosis: "EventBridge rule not configured, SNS subscription not confirmed, or email in spam."
  resolution: "Create EventBridge rule → SNS topic → email subscription. Confirm subscription."

- symptoms: "Notifications delayed"
  diagnosis: "Health events have inherent delay. EventBridge adds minimal additional latency."
  resolution: "Expected behavior. Health events are not real-time. Use CloudWatch for real-time alerts."

- symptoms: "Too many notifications"
  diagnosis: "EventBridge rule matching all health events including informational ones."
  resolution: "Filter by eventTypeCategory in rule pattern. Focus on 'issue' and 'scheduledChange'."

## Output Format

```yaml
root_cause: "notification_setup — <specific_cause>"
evidence:
  - type: eventbridge_rule
    content: "<rule configuration and targets>"
  - type: sns_config
    content: "<SNS topic and subscription status>"
severity: MEDIUM
mitigation:
  immediate: "Fix EventBridge rule, SNS permissions, or subscription"
  long_term: "Implement multi-channel notification with severity filtering"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never dismiss or archive health events without documenting their resolution
- Never disable EventBridge rules for health notifications in production accounts
- Never ignore scheduled maintenance events without planning for impact
- Never delete organizational health view configurations without stakeholder approval
- Never modify health event filters that may suppress critical notifications

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable EventBridge notification rules if health alerts stop flowing
- Restore previous event filter configurations if critical events are missed
- Revert organizational view delegation changes if visibility is lost

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
