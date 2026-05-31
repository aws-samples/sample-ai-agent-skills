# AWS Data Exchange Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Data Exchange problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Data Exchange — dataset creation, revision publishing, subscriptions, entitlements, S3 exports, API assets, Redshift query access, Lake Formation access, provider configuration, subscriber management, and event notifications.

### Activate When

- Dataset creation failures
- Revision publishing errors
- Subscription issues
- Entitlement problems
- S3 export failures
- API asset errors
- Redshift query access issues
- Lake Formation access problems
- Provider configuration errors
- Subscriber management issues
- Event notification failures
- General Data Exchange errors

---

## Skill Structure

```
dataexchange-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-dataset-creation-failures.md
    ├── A2-revision-publishing.md
    ├── B1-subscription-errors.md
    ├── B2-entitlement-issues.md
    ├── C1-s3-export-failures.md
    ├── C2-api-asset-errors.md
    ├── D1-redshift-query-access.md
    ├── D2-lake-formation-access.md
    ├── E1-provider-configuration.md
    ├── E2-subscriber-management.md
    ├── F1-event-notifications.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Data Sets** | A1–A2 | Creation, revision publishing |
| **B — Subscriptions** | B1–B2 | Subscription errors, entitlements |
| **C — Exports** | C1–C2 | S3 exports, API assets |
| **D — Data Access** | D1–D2 | Redshift query, Lake Formation |
| **E — Provider/Subscriber** | E1–E2 | Provider config, subscriber management |
| **F — Events** | F1 | Event notifications |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `dataexchange:*`, `s3:GetObject`, `s3:PutObject`, `redshift:GetClusterCredentials`, `lakeformation:*`, `ram:*` permissions

---

## License

MIT-0
