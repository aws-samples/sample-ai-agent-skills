---
name: chatbot-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Chatbot problems by
  analyzing Slack workspace configurations, Microsoft Teams channel setups,
  SNS notification delivery, EventBridge integration, CLI commands in chat,
  command permissions, channel IAM roles, guardrail policies, CloudWatch
  alarm notifications, and Security Hub finding delivery using structured
  runbooks. Activate when: Slack workspace configuration issues, Teams
  channel setup failures, SNS notification not delivered to chat, EventBridge
  events not appearing, CLI commands failing in chat, permission errors for
  chat commands, channel role misconfiguration, guardrail policy blocking
  actions, CloudWatch alarm notifications missing, Security Hub integration
  not working, or the user says something is wrong with AWS Chatbot without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Chatbot, SNS, EventBridge, IAM,
  CloudWatch, Security Hub, CloudTrail, and optionally Lambda permissions.
  Slack workspace admin access or Microsoft Teams admin access for
  channel configuration.
---

# AWS Chatbot Diagnostics

## When to use

Any AWS Chatbot investigation where the console alone is insufficient — Slack/Teams configuration, notification delivery debugging, CLI command troubleshooting, IAM role and guardrail policy issues, or integration problems with CloudWatch and Security Hub.

## Investigation workflow

### Step 1 — Collect and triage

```
aws chatbot describe-slack-workspaces
aws chatbot describe-slack-channel-configurations
aws chatbot describe-chime-webhook-configurations
aws chatbot list-microsoft-teams-configured-teams
aws chatbot list-microsoft-teams-channel-configurations
```

### Step 2 — Domain deep dive

```
aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
aws sns get-topic-attributes --topic-arn <topic-arn>
aws iam get-role --role-name <chatbot-role-name>
aws iam list-attached-role-policies --role-name <chatbot-role-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 20
aws cloudwatch describe-alarms --alarm-names <alarm-name> --query 'MetricAlarms[0].AlarmActions'
aws events list-rules --query 'Rules[*].{Name:Name,State:State}'
version: "1.0.0"
last_updated: "2025-04-12"
aws securityhub get-findings --filters '{"ProductName":[{"Value":"Chatbot","Comparison":"EQUALS"}]}'
```

Read `references/guardrails.md` before concluding on any AWS Chatbot issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `chatbot describe-slack-workspaces` | List configured Slack workspaces |
| `chatbot describe-slack-channel-configurations` | Slack channel details |
| `chatbot list-microsoft-teams-channel-configurations` | Teams channel details |
| `sns list-subscriptions-by-topic` | SNS subscriptions for notification topics |
| `sns get-topic-attributes` | SNS topic configuration |
| `iam get-role` | Channel IAM role details |
| `iam list-attached-role-policies` | Policies on channel role |
| `cloudtrail lookup-events` | Recent Chatbot API events |
| `cloudwatch describe-alarms` | Alarm action configuration |
| `events list-rules` | EventBridge rules for Chatbot |

## Gotchas: AWS Chatbot

- Chatbot requires workspace/team-level authorization first — you must authorize the Slack workspace or Teams tenant in the AWS Console before configuring channels. This is a one-time manual step.
- Channel configurations are per-channel, per-account — each Slack channel or Teams channel needs its own configuration with an IAM role and SNS topic subscriptions.
- IAM channel role controls what CLI commands can do — the role attached to the channel configuration determines what AWS actions can be performed via chat commands. This is separate from notification delivery.
- Guardrail policies restrict CLI command actions — guardrail policies (IAM policies) limit what actions can be executed via chat, even if the channel role has broader permissions. They act as an additional permission boundary.
- SNS topics must be subscribed to the Chatbot endpoint — notifications flow through SNS. The Chatbot channel must be subscribed to the SNS topic. Missing subscriptions mean no notifications.
- Not all AWS services send notifications through Chatbot — Chatbot supports specific notification types: CloudWatch Alarms, AWS Health events, Security Hub findings, Budgets alerts, and custom EventBridge events. Not every AWS service integrates directly.
- CLI commands in chat use a specific syntax — commands must start with `@aws` followed by the CLI command. Not all CLI commands are supported. Read-only commands are allowed by default; mutating commands require explicit guardrail policy approval.
- Slack app must be installed in the workspace — the AWS Chatbot Slack app must be installed and authorized. Removing or restricting the app breaks all channel configurations.
- Teams requires tenant-level consent — Microsoft Teams integration requires Azure AD tenant admin consent for the AWS Chatbot app.
- Notification formatting is Chatbot-managed — you cannot customize the notification card format. Chatbot formats notifications based on the source service.
- Rate limits apply to notifications — Chatbot has rate limits on notification delivery. High-volume alarm states can be throttled.
- EventBridge integration requires specific event patterns — not all EventBridge events are supported. Chatbot processes events matching specific patterns.

## Anti-hallucination rules

1. Always cite specific channel configurations, IAM policies, or SNS subscriptions as evidence. Never guess at Chatbot settings.
2. Chatbot requires workspace/team authorization in the console. Never suggest this can be done via CLI alone.
3. Guardrail policies are separate from channel IAM roles. Never conflate the two permission layers.
4. Not all AWS services integrate with Chatbot. Never suggest Chatbot can deliver notifications from unsupported services.
5. CLI commands in chat have specific syntax and restrictions. Never suggest arbitrary CLI commands work in chat.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Configuration | A1-A2 | Slack workspace, Teams channel |
| B — Notifications | B1-B2 | SNS notifications, EventBridge |
| C — Commands | C1-C2 | CLI commands in chat, permissions |
| D — IAM | D1-D2 | Channel role, guardrail policies |
| E — Integration | E1-E2 | CloudWatch alarms, Security Hub |
| Z — Catch-All | Z1 | General troubleshooting |
