# AWS Chatbot Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Chatbot problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Chatbot when the console alone isn't enough — Slack/Teams configuration, notification delivery debugging, CLI command troubleshooting, IAM role and guardrail policy issues, and integration problems with CloudWatch and Security Hub.

### Activate When

- Slack workspace authorization or configuration failures
- Microsoft Teams channel setup issues
- SNS notifications not delivered to chat channels
- EventBridge events not appearing in chat
- CLI commands failing or returning errors in chat
- Permission errors when running commands in chat
- Channel IAM role misconfiguration
- Guardrail policy blocking legitimate actions
- CloudWatch alarm notifications not appearing
- Security Hub finding notifications missing

---

## Skill Structure

```
chatbot-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-slack-workspace.md
    ├── A2-teams-channel.md
    ├── B1-sns-notifications.md
    ├── B2-eventbridge.md
    ├── C1-cli-commands.md
    ├── C2-command-permissions.md
    ├── D1-channel-role.md
    ├── D2-guardrail-policies.md
    ├── E1-cloudwatch-alarms.md
    ├── E2-security-hub.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (16 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Configuration** | A1–A2 | Slack workspace, Teams channel |
| **B — Notifications** | B1–B2 | SNS notifications, EventBridge |
| **C — Commands** | C1–C2 | CLI commands in chat, permissions |
| **D — IAM** | D1–D2 | Channel role, guardrail policies |
| **E — Integration** | E1–E2 | CloudWatch alarms, Security Hub |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering workspace authorization requirements, channel configuration scope, IAM role vs guardrail policy separation, SNS subscription requirements, supported notification types, CLI command syntax and restrictions, Slack app installation, Teams tenant consent, notification formatting, rate limits, EventBridge event patterns, and read-only vs mutating command defaults.

---

## Investigation Workflow

1. **Triage** — List workspaces/teams, describe channel configurations, check SNS subscriptions
2. **Deep Dive** — Examine channel IAM role, guardrail policies, SNS topic attributes
3. **Detailed** — Review CloudTrail events, check alarm actions, verify EventBridge rules

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `chatbot:*`, `sns:ListSubscriptionsByTopic`, `sns:GetTopicAttributes`, `iam:GetRole`, `iam:ListAttachedRolePolicies`, `cloudwatch:DescribeAlarms`, `events:ListRules`, `securityhub:GetFindings`, `cloudtrail:LookupEvents`
- Slack workspace admin access or Microsoft Teams admin access

---

## Usage Examples

```
# List Slack workspaces
aws chatbot describe-slack-workspaces

# List Slack channel configurations
aws chatbot describe-slack-channel-configurations

# List Teams configurations
aws chatbot list-microsoft-teams-channel-configurations

# Check SNS subscriptions
aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic

# Check channel IAM role
aws iam get-role --role-name AWSChatbotRole
```

---

## License

MIT-0
