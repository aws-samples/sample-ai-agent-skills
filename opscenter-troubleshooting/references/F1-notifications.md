---
title: "F1 — Notifications"
description: "Diagnose OpsItem notification delivery issues"
status: active
severity: MEDIUM
triggers: ["no notifications", "not notified", "OpsItem alert missing"]
owner: devops-agent
objective: "Resolve OpsItem notification delivery"
context: "OpsCenter has no built-in notifications. Notifications require EventBridge rules routing OpsItem events to SNS topics."
---

## Phase 1 — Triage

MUST:
- Check for EventBridge rules matching OpsItem events: `aws events list-rules`
- Verify SNS topic exists and has subscribers: `aws sns list-subscriptions-by-topic --topic-arn <arn>`
- Check EventBridge rule targets include SNS

SHOULD:
- Verify SNS subscriptions are confirmed
- Check SNS delivery logs for failures

MAY:
- Test notification pipeline: create test OpsItem and verify notification

## Phase 2 — Remediate

MUST:
- Create EventBridge rule matching OpsItem creation/update events
- Set SNS topic as rule target
- Subscribe notification recipients to SNS topic

SHOULD:
- Confirm all SNS subscriptions
- Filter notifications by severity or source

MAY:
- Add multiple notification channels (email, Slack via Chatbot)

## Common Issues

- symptoms: "No notifications when critical OpsItems are created"
  diagnosis: "No EventBridge rule configured for OpsItem events."
  resolution: "Create EventBridge rule: source=aws.ssm, detail-type=OpsItem Create → SNS topic."

## Output Format

```yaml
root_cause: "notifications — <specific_cause>"
evidence:
  - type: notification_config
    content: "<EventBridge rule and SNS configuration>"
severity: MEDIUM
mitigation:
  immediate: "Configure notification pipeline"
  long_term: "Monitor notification delivery"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — notifications may contain operational details
- Service Disruption Risk: LOW — notification failures do not affect resources
- Compliance Impact: MEDIUM — missed notifications can delay incident response

## Escalation Conditions
- All OpsCenter notifications stop being delivered
- Notification delivery to critical channels fails
- SNS topic permissions prevent notification delivery
- Notification content contains sensitive data
- Notification volume causes alert fatigue

## Data Sensitivity
- Notification content — INTERNAL (may contain CONFIDENTIAL data)
- SNS topic ARNs and configurations — INTERNAL
- Subscriber email addresses — CONFIDENTIAL
- Notification routing rules — INTERNAL
- Never include credentials or PII in notification content

## Prohibited Actions
- Do NOT disable notifications for critical OpsItems
- Do NOT send notifications to unauthorized recipients
- Do NOT include sensitive data in notification messages
- Do NOT ignore notification delivery failures
- Do NOT create notification rules that cause excessive alert volume
- Do NOT share SNS topic access with unauthorized accounts

## Phase 3 — Rollback
- Restore previous notification configurations
- Re-enable disabled notification rules
- Restore SNS topic permissions if modified
- Verify notification delivery after changes
- Remove unauthorized notification subscribers
- Document notification configuration changes

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
