# AWS SSM OpsCenter Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Systems Manager OpsCenter problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for SSM OpsCenter — OpsItem creation and management, automation runbooks, related resources, deduplication, EventBridge integration, and notifications.

### Activate When

- OpsItem creation failures
- Automation runbook execution errors
- Related resource association issues
- Deduplication not working as expected
- EventBridge rules not creating OpsItems
- Notification delivery failures
- General OpsCenter errors

---

## Skill Structure

```
opscenter-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-opsitem-creation.md
    ├── A2-opsitem-management.md
    ├── B1-automation-runbooks.md
    ├── B2-runbook-execution.md
    ├── C1-related-resources.md
    ├── C2-resource-association.md
    ├── D1-deduplication.md
    ├── D2-dedup-configuration.md
    ├── E1-eventbridge-integration.md
    ├── E2-event-rules.md
    ├── F1-notifications.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — OpsItems** | A1–A2 | Creation, management |
| **B — Automation** | B1–B2 | Runbooks, execution |
| **C — Resources** | C1–C2 | Related resources, association |
| **D — Deduplication** | D1–D2 | Behavior, configuration |
| **E — EventBridge** | E1–E2 | Integration, event rules |
| **F — Notifications** | F1 | Delivery |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `ssm:*`, `events:*`, `sns:*`, `iam:Get*` permissions

---

## License

MIT-0
