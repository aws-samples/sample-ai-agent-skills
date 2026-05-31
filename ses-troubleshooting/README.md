# SES Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon SES problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon SES when the console alone isn't enough — send failures, bounce and complaint handling, deliverability and reputation issues, DKIM/SPF/DMARC authentication, suppression list management, email receiving, templates, configuration sets, sandbox limitations, SMTP credentials, and sending quotas.

### Activate When

- Emails failing to send or being rejected
- High bounce rate or bounce notifications
- Complaint rate increasing
- Reputation dashboard showing warnings
- DKIM, SPF, or DMARC authentication failures
- Addresses stuck on suppression list
- Receipt rules not processing incoming email
- S3 action failing for received emails
- Email template rendering errors
- Personalization data not substituting
- Configuration set event destinations not working
- Stuck in sandbox mode
- Production access request denied
- IAM permission errors for SES operations
- SMTP authentication failures
- Sending quota exceeded
- Rate throttling errors

---

## Skill Structure

```
ses-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-send-failures.md
    ├── A2-bounce-handling.md
    ├── A3-complaint-handling.md
    ├── B1-reputation.md
    ├── B2-dkim-spf-dmarc.md
    ├── B3-suppression-list.md
    ├── C1-receipt-rules.md
    ├── C2-s3-actions.md
    ├── D1-template-errors.md
    ├── D2-personalization.md
    ├── E1-configuration-sets.md
    ├── E2-event-destinations.md
    ├── F1-sandbox-limitations.md
    ├── F2-production-access.md
    ├── G1-iam-permissions.md
    ├── G2-smtp-credentials.md
    ├── H1-sending-limits.md
    ├── H2-rate-throttling.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Sending** | A1–A3 | Send failures, bounce handling, complaint handling |
| **B — Deliverability** | B1–B3 | Reputation, DKIM/SPF/DMARC, suppression list |
| **C — Receiving** | C1–C2 | Receipt rules, S3 actions |
| **D — Templates** | D1–D2 | Template errors, personalization |
| **E — Configuration** | E1–E2 | Configuration sets, event destinations |
| **F — Sandbox** | F1–F2 | Sandbox limitations, production access |
| **G — Security** | G1–G2 | IAM permissions, SMTP credentials |
| **H — Quotas** | H1–H2 | Sending limits, rate throttling |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
