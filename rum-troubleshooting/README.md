# Amazon CloudWatch RUM Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CloudWatch RUM (Real User Monitoring) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudWatch RUM — app monitor creation, configuration, data collection, JavaScript snippet issues, custom events, page performance analysis, session replay, user journey analysis, Cognito integration, CloudWatch metric export, and data retention.

### Activate When

- App monitor creation failures
- App monitor configuration issues
- Data collection failures
- JavaScript snippet problems
- Custom event errors
- Page performance analysis issues
- Session replay failures
- User journey analysis problems
- Cognito integration issues
- CloudWatch metric export failures
- Data retention configuration issues
- General RUM errors

---

## Skill Structure

```
rum-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-app-monitor-creation.md
    ├── A2-app-monitor-configuration.md
    ├── B1-data-collection-failures.md
    ├── B2-javascript-snippet-issues.md
    ├── C1-custom-event-errors.md
    ├── C2-page-performance-analysis.md
    ├── D1-session-replay-issues.md
    ├── D2-user-journey-analysis.md
    ├── E1-cognito-integration.md
    ├── E2-cloudwatch-metric-export.md
    ├── F1-data-retention-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — App Monitor Setup** | A1–A2 | Creation, configuration |
| **B — Data Collection** | B1–B2 | Collection failures, JavaScript snippet issues |
| **C — Events & Performance** | C1–C2 | Custom event errors, page performance analysis |
| **D — Session Analysis** | D1–D2 | Session replay issues, user journey analysis |
| **E — Integrations** | E1–E2 | Cognito integration, CloudWatch metric export |
| **F — Data Management** | F1 | Data retention configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `rum:*`, `cognito-identity:*`, `cloudwatch:Get*`, `iam:Get*`, `logs:Describe*` permissions

---

## License

MIT-0
