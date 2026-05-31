---
name: chatbot-advanced-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot advanced AWS Chatbot
  problems by analyzing custom action failures, action approvals,
  Microsoft Teams integration, guardrail configuration, notification
  filtering, run command from chat, Lambda invocation from chat,
  cross-account notifications, event pattern matching, and audit logging.
  Activate when: custom action failures, approval issues, Teams
  integration problems, guardrail configuration errors, notification
  filtering issues, run command from chat failures, Lambda invocation
  errors, cross-account notification problems, event pattern matching
  issues, audit logging gaps, or the user says something is wrong
  with advanced AWS Chatbot features.
compatibility: >
  Requires AWS CLI or SDK access with chatbot, sns, iam, lambda,
  ssm, cloudtrail, cloudwatch, and organizations permissions.
---

# AWS Chatbot Advanced Diagnostics

## When to use

Any advanced AWS Chatbot investigation — custom actions, action approvals, Microsoft Teams integration, guardrail configuration, notification filtering, run command from chat, Lambda invocation, cross-account notifications, event pattern matching, or audit logging.

## Investigation workflow

### Step 1 — Collect and triage

```
aws chatbot describe-slack-channel-configurations --max-results 20
aws chatbot list-microsoft-teams-channel-configurations --max-results 20
aws sns list-topics --query 'Topics[*].TopicArn'
```

### Step 2 — Domain deep dive

```
aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>
aws chatbot get-microsoft-teams-channel-configuration --chat-configuration-arn <arn>
aws iam get-role --role-name <chatbot-role-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name /aws/chatbot/<configuration-name> --start-time <timestamp>
aws sns get-topic-attributes --topic-arn <topic-arn>
```

Read `references/guardrails.md` before concluding on any advanced Chatbot issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `chatbot describe-slack-channel-configurations` | List Slack configurations |
| `chatbot list-microsoft-teams-channel-configurations` | List Teams configurations |
| `chatbot get-microsoft-teams-channel-configuration` | Get Teams config details |
| `sns list-topics` | List SNS topics for notifications |
| `sns get-topic-attributes` | Get topic configuration |
| `iam get-role` | Check Chatbot IAM role permissions |
| `cloudtrail lookup-events` | Audit Chatbot actions |

## Gotchas: AWS Chatbot Advanced

- Custom actions allow running predefined commands from chat. They require explicit IAM permissions on the Chatbot channel role AND the action must be defined in the channel configuration.
- Microsoft Teams integration uses a different OAuth flow than Slack. Teams requires Azure AD app registration and Chatbot workspace configuration before channel setup.
- Guardrails restrict which commands and actions can be run from chat. They are configured per channel and use IAM policy syntax. Overly restrictive guardrails silently block commands.
- Run Command from chat executes SSM Run Command on instances. It requires the Chatbot role to have ssm:SendCommand AND the target instances must have SSM agent running.
- Cross-account notifications require the SNS topic in the source account to allow the Chatbot account to subscribe. The Chatbot configuration must reference the cross-account topic ARN.
- Event pattern matching for notifications uses EventBridge patterns. Incorrect patterns cause notifications to be silently dropped — no error is generated.
- Audit logging for Chatbot actions goes to CloudTrail. Chat commands and their results are logged, but the chat message content itself is not stored in CloudTrail.

## Anti-hallucination rules

1. Always cite specific configuration ARNs, channel IDs, or API responses as evidence.
2. Slack and Teams integrations have different setup flows and limitations. Never conflate them.
3. Guardrails use IAM policy syntax but are NOT IAM policies. Never suggest modifying IAM policies to fix guardrail issues.
4. Run Command from chat and Lambda invocation from chat have different permission models. Never mix them up.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Custom Actions | A1-A2 | Custom action failures, action approval issues |
| B — Teams | B1-B2 | Teams integration, Teams channel config |
| C — Guardrails | C1-C2 | Guardrail configuration, notification filtering |
| D — Operations | D1-D2 | Run command from chat, Lambda invocation from chat |
| E — Cross-Account | E1-E2 | Cross-account notifications, event pattern matching |
| F — Audit | F1 | Audit logging |
| Z — Catch-All | Z1 | General troubleshooting |
