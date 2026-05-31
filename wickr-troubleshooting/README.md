# AWS Wickr Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Wickr problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Wickr — network creation, admin console access, user provisioning, SSO integration, message delivery, file sharing, retention policies, compliance logging, bot integration, API access, and federation.

### Activate When

- Network creation failures
- Admin console access issues
- User provisioning errors
- SSO integration problems
- Message delivery failures
- File sharing issues
- Retention policy configuration errors
- Compliance logging problems
- Bot integration failures
- API access issues
- Federation problems
- General Wickr errors

---

## Skill Structure

```
wickr-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-network-creation.md
    ├── A2-admin-console-access.md
    ├── B1-user-provisioning.md
    ├── B2-sso-integration.md
    ├── C1-message-delivery-failures.md
    ├── C2-file-sharing-issues.md
    ├── D1-retention-policy-configuration.md
    ├── D2-compliance-logging.md
    ├── E1-bot-integration.md
    ├── E2-api-access.md
    ├── F1-federation-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Network & Admin** | A1–A2 | Network creation, admin console access |
| **B — Users & Auth** | B1–B2 | User provisioning, SSO integration |
| **C — Messaging** | C1–C2 | Message delivery, file sharing |
| **D — Compliance** | D1–D2 | Retention policies, compliance logging |
| **E — Integration** | E1–E2 | Bot integration, API access |
| **F — Federation** | F1 | Federation issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `wickr:*`, `iam:Get*`, `s3:GetBucket*`, `kms:Describe*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
