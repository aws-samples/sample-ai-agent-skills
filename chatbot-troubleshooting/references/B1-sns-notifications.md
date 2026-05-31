---
title: "B1 — AWS Chatbot SNS Notification Issues"
description: "Diagnose SNS notification delivery failures to Chatbot channels"
status: active
severity: HIGH
triggers:
  - "notification not delivered"
  - "SNS notification"
  - "no alerts in Slack"
  - "no alerts in Teams"
  - "missing notification"
owner: devops-agent
objective: "Identify and resolve SNS notification delivery issues to Chatbot channels"
context: "All Chatbot notifications flow through SNS topics. The channel must be subscribed to the SNS topic, and the topic must receive events from the source service. Missing subscriptions or topic misconfigurations cause notification failures."
---

## Phase 1 — Triage

MUST:
- Check channel SNS topics: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>` (look at snsTopicArns)
- Check SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Verify subscription is confirmed (not PendingConfirmation)
- Check SNS topic attributes: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Verify the source service is publishing to the SNS topic

SHOULD:
- Check SNS delivery status logging: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.{HTTPSuccessFeedbackRoleArn:HTTPSuccessFeedbackRoleArn}'`
- Verify SNS topic policy allows the source service to publish
- Check for SNS topic encryption that may block delivery

MAY:
- Check CloudWatch metrics for SNS: NumberOfMessagesPublished, NumberOfNotificationsDelivered
- Verify no SNS filter policy is blocking messages

## Phase 2 — Remediate

MUST:
- Add SNS topic ARN to the channel configuration if missing
- Ensure subscription is confirmed (Chatbot auto-confirms)
- Verify source service alarm/event actions include the SNS topic ARN

SHOULD:
- Check SNS topic policy allows publishing from the source service
- Remove encryption if it blocks Chatbot delivery (or grant KMS permissions)
- Test by publishing a test message: `aws sns publish --topic-arn <topic-arn> --message "Test notification"`

MAY:
- Enable SNS delivery status logging for debugging
- Set up CloudWatch alarms on SNS delivery failures

## Common Issues

- symptoms: "No notifications in Slack/Teams despite alarms firing"
  diagnosis: "SNS topic not included in the channel configuration."
  resolution: "Add the SNS topic ARN to the Chatbot channel configuration."

- symptoms: "SNS topic configured but subscription missing"
  diagnosis: "Channel configuration doesn't create subscription automatically."
  resolution: "Verify subscription exists. Recreate channel configuration if needed."

- symptoms: "Notifications were working but stopped"
  diagnosis: "SNS topic policy changed or KMS key permissions revoked."
  resolution: "Check topic policy and KMS key permissions."

## Output Format

```yaml
root_cause: "sns_notification_issue — <specific_cause>"
evidence:
  - type: channel_topics
    content: "<configured SNS topic ARNs>"
  - type: subscriptions
    content: "<SNS subscription status>"
  - type: topic_policy
    content: "<SNS topic access policy>"
severity: HIGH
mitigation:
  immediate: "Fix SNS topic configuration or subscription"
  long_term: "Implement notification delivery monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Add SNS topic ARN to the channel configuration if missing | GREEN |
| Ensure subscription is confirmed | GREEN |
| Verify source service alarm/event actions include the SNS topic ARN | GREEN |
| Check SNS topic policy allows publishing from the source service | YELLOW |
| Remove encryption if it blocks Chatbot delivery (or grant KMS permissions) | YELLOW |
| Test by publishing a test message | GREEN |
| Enable SNS delivery status logging for debugging | GREEN |
| Set up CloudWatch alarms on SNS delivery failures | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- SNS messages may contain alarm details, security findings, or operational data

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If SNS topic was added to channel configuration, remove it if it was incorrect
2. If SNS topic policy was modified, restore the previous policy
3. If KMS permissions were changed, revert to the previous key policy
4. If encryption was removed from the SNS topic, re-enable it
5. Verify rollback by confirming SNS topic and subscription configuration matches expected state

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
