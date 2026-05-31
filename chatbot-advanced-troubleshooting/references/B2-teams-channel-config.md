---
title: "B2 — Teams Channel Config"
description: "Diagnose Microsoft Teams channel configuration issues including SNS topics, IAM roles, and notification delivery"
status: active
severity: MEDIUM
triggers: ["Teams channel config error", "Teams notifications not working", "Teams channel setup", "Teams SNS topic"]
owner: devops-agent
objective: "Resolve Microsoft Teams channel configuration and notification delivery issues"
context: "Teams channel configurations link a specific Teams channel to SNS topics and an IAM role. Each configuration has its own guardrails, notification preferences, and logging settings."
---

## Phase 1 — Triage

MUST:
- Get channel configuration: `aws chatbot get-microsoft-teams-channel-configuration --chat-configuration-arn <arn>`
- Verify SNS topic subscriptions: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`
- Check the channel IAM role: `aws iam get-role --role-name <role-name>`

SHOULD:
- Verify the Teams channel ID is correct in the configuration
- Check SNS topic delivery status
- Review notification preferences in the configuration

MAY:
- Test notification delivery with a test SNS message: `aws sns publish --topic-arn <arn> --message "test"`
- Check Chatbot logs for delivery failures

## Phase 2 — Remediate

MUST:
- Fix channel configuration with correct Teams channel ID and team ID
- Ensure SNS topics are correctly associated with the configuration
- Verify the IAM role has required permissions

SHOULD:
- Configure notification preferences to filter relevant events
- Set up logging for notification delivery troubleshooting
- Test notification delivery after configuration changes

MAY:
- Create separate configurations for different notification types
- Implement notification formatting customization

## Safety Ratings

```
safety_ratings:
  - "get-microsoft-teams-channel-configuration: GREEN — read-only configuration"
  - "sns list-subscriptions-by-topic: GREEN — read-only subscription listing"
  - "update-microsoft-teams-channel-configuration: YELLOW — modifies channel settings"
  - "sns publish: YELLOW — sends test message to topic subscribers"
```

## Escalation Conditions

- Channel configuration changes affect production notification delivery
- SNS topic policy changes required for cross-account scenarios
- Teams channel permissions prevent Chatbot from posting
- Multiple channel configurations need simultaneous updates

## Data Sensitivity

- `aws chatbot get-microsoft-teams-channel-configuration`: LOW — configuration metadata
- `aws sns list-subscriptions-by-topic`: LOW — subscription endpoints
- `aws iam get-role`: LOW — role metadata
- Notification content: MEDIUM — may contain AWS resource details

## Prohibited Actions

- NEVER delete a production Teams channel configuration without backup
- NEVER modify SNS topic policies without understanding all subscribers
- NEVER share Teams channel webhook URLs or configuration details externally
- NEVER disable notification filtering to troubleshoot — it floods the channel

## Phase 3 — Rollback

1. If configuration was changed incorrectly, update with previous settings
2. If SNS subscriptions were modified, restore previous subscription state
3. If the IAM role was changed, revert permissions
4. Verify notifications resume after rollback
5. Notify channel users about temporary notification disruption

## Common Issues

- symptoms: "Teams channel receives no notifications despite correct configuration"
  diagnosis: "SNS subscription is not confirmed, or the topic has no events published to it."
  resolution: "Check SNS subscription status. Verify events are being published to the topic. Confirm the subscription."

- symptoms: "Notifications appear in wrong Teams channel"
  diagnosis: "The channel ID in the configuration points to a different channel."
  resolution: "Update the configuration with the correct Teams channel ID. Verify using Teams channel properties."

- symptoms: "Notification formatting is broken in Teams"
  diagnosis: "The notification source sends Slack-formatted messages that do not render in Teams."
  resolution: "Teams uses Adaptive Cards format. Ensure the notification source formats for the correct platform."

## Output Format

```yaml
root_cause: "teams_channel_config — <specific_cause>"
evidence:
  - type: config_error
    content: "<channel configuration details and issue>"
severity: MEDIUM
mitigation:
  immediate: "Fix channel configuration"
  long_term: "Implement configuration validation and monitoring"
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
