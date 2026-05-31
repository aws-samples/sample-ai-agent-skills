---
title: "E1 — AWS Chatbot CloudWatch Alarm Notification Issues"
description: "Diagnose CloudWatch alarm notification delivery to Chatbot channels"
status: active
severity: HIGH
triggers:
  - "CloudWatch alarm notification"
  - "alarm not in Slack"
  - "alarm not in Teams"
  - "alarm notification missing"
  - "alarm action"
owner: devops-agent
objective: "Identify and resolve CloudWatch alarm notification delivery issues to Chatbot channels"
context: "CloudWatch alarms send notifications to SNS topics, which Chatbot subscribes to. The alarm must have the SNS topic as an action, and the Chatbot channel must be configured with that topic."
---

## Phase 1 — Triage

MUST:
- Check alarm actions: `aws cloudwatch describe-alarms --alarm-names <alarm-name> --query 'MetricAlarms[0].{AlarmActions:AlarmActions,OKActions:OKActions,InsufficientDataActions:InsufficientDataActions}'`
- Verify SNS topic is in alarm actions
- Check SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify Chatbot channel includes the SNS topic: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>`
- Check alarm state: `aws cloudwatch describe-alarms --alarm-names <alarm-name> --query 'MetricAlarms[0].StateValue'`

SHOULD:
- Verify alarm actually transitioned state (check state history)
- Check SNS delivery logs for failures
- Verify no SNS filter policy is blocking alarm notifications

MAY:
- Test with a manual alarm state change: `aws cloudwatch set-alarm-state --alarm-name <name> --state-value ALARM --state-reason "Testing"`
- Check CloudWatch alarm history: `aws cloudwatch describe-alarm-history --alarm-name <alarm-name>`

## Phase 2 — Remediate

MUST:
- Add SNS topic ARN to alarm actions (AlarmActions, OKActions as needed)
- Ensure SNS topic is in the Chatbot channel configuration
- Verify SNS subscription to Chatbot is confirmed

SHOULD:
- Configure both ALARM and OK actions for state change notifications
- Test notification delivery with a manual state change
- Monitor SNS delivery metrics

MAY:
- Set up composite alarms for complex notification logic
- Create separate SNS topics for different severity levels

## Common Issues

- symptoms: "Alarm fires but no notification in Slack/Teams"
  diagnosis: "Alarm actions don't include the SNS topic configured in Chatbot."
  resolution: "Add the Chatbot SNS topic ARN to the alarm's AlarmActions."

- symptoms: "SNS topic in alarm actions but no Chatbot notification"
  diagnosis: "Chatbot channel not subscribed to the SNS topic."
  resolution: "Add the SNS topic to the Chatbot channel configuration."

- symptoms: "OK notifications received but not ALARM notifications"
  diagnosis: "Only OKActions configured, not AlarmActions."
  resolution: "Add the SNS topic to AlarmActions as well."

## Output Format

```yaml
root_cause: "cloudwatch_alarm_notification_issue — <specific_cause>"
evidence:
  - type: alarm_actions
    content: "<alarm action configuration>"
  - type: sns_subscriptions
    content: "<SNS topic subscriptions>"
  - type: channel_config
    content: "<Chatbot channel SNS topics>"
severity: HIGH
mitigation:
  immediate: "Fix alarm actions or SNS topic configuration"
  long_term: "Standardize alarm notification configuration with IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add SNS topic ARN to alarm actions | GREEN |
| Ensure SNS topic is in the Chatbot channel configuration | GREEN |
| Verify SNS subscription to Chatbot is confirmed | GREEN |
| Configure both ALARM and OK actions for state change notifications | GREEN |
| Test notification delivery with a manual state change | GREEN |
| Monitor SNS delivery metrics | GREEN |
| Set up composite alarms for complex notification logic | YELLOW |
| Create separate SNS topics for different severity levels | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- CloudWatch alarm notifications may contain metric values and resource identifiers

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If alarm actions were modified, restore the previous alarm action configuration: `aws cloudwatch put-metric-alarm` with previous settings
2. If SNS topics were added to channel configuration, remove them if incorrect
3. If composite alarms were created, delete them: `aws cloudwatch delete-alarms --alarm-names <name>`
4. Verify rollback by confirming alarm actions and SNS configuration match expected state

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
