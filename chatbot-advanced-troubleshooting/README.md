# AWS Chatbot Advanced Diagnostics Skill

Agent skill for investigating and troubleshooting advanced AWS Chatbot problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for advanced AWS Chatbot — custom actions, action approvals, Microsoft Teams integration, guardrail configuration, notification filtering, run command from chat, Lambda invocation, cross-account notifications, event pattern matching, and audit logging.

### Activate When

- Custom action failures
- Action approval issues
- Microsoft Teams integration problems
- Teams channel configuration errors
- Guardrail configuration issues
- Notification filtering problems
- Run command from chat failures
- Lambda invocation from chat errors
- Cross-account notification issues
- Event pattern matching failures
- Audit logging gaps
- General advanced Chatbot errors

---

## Skill Structure

```
chatbot-advanced-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-custom-action-failures.md
    ├── A2-action-approval-issues.md
    ├── B1-teams-integration.md
    ├── B2-teams-channel-config.md
    ├── C1-guardrail-configuration.md
    ├── C2-notification-filtering.md
    ├── D1-run-command-from-chat.md
    ├── D2-lambda-invocation-from-chat.md
    ├── E1-cross-account-notifications.md
    ├── E2-event-pattern-matching.md
    ├── F1-audit-logging.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Custom Actions** | A1–A2 | Action failures, approval issues |
| **B — Teams** | B1–B2 | Integration, channel config |
| **C — Guardrails** | C1–C2 | Configuration, notification filtering |
| **D — Operations** | D1–D2 | Run command, Lambda invocation |
| **E — Cross-Account** | E1–E2 | Notifications, event patterns |
| **F — Audit** | F1 | Audit logging |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `chatbot:*`, `sns:*`, `iam:Get*`, `lambda:InvokeFunction`, `ssm:SendCommand`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
