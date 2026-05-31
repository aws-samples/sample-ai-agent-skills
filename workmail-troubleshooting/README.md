# Amazon WorkMail Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon WorkMail problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon WorkMail — organization creation, domain verification, email delivery, receiving issues, client connectivity, mobile sync, mailbox management, group management, Exchange migration, journaling, and access control.

### Activate When

- Organization creation failures
- Domain verification issues
- Email delivery failures
- Email receiving problems
- Client connectivity errors
- Mobile device sync issues
- User mailbox problems
- Group management issues
- Exchange migration failures
- Journaling rule errors
- Access control rule issues
- General WorkMail errors

---

## Skill Structure

```
workmail-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-organization-creation.md
    ├── A2-domain-verification.md
    ├── B1-email-delivery-failures.md
    ├── B2-receiving-issues.md
    ├── C1-client-connectivity.md
    ├── C2-mobile-device-sync.md
    ├── D1-user-mailbox-issues.md
    ├── D2-group-management.md
    ├── E1-migration-from-exchange.md
    ├── E2-journaling-rules.md
    ├── F1-access-control-rules.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Organization** | A1–A2 | Creation, domain verification |
| **B — Email Flow** | B1–B2 | Delivery failures, receiving issues |
| **C — Clients** | C1–C2 | Client connectivity, mobile device sync |
| **D — Mailboxes** | D1–D2 | User mailbox issues, group management |
| **E — Operations** | E1–E2 | Exchange migration, journaling rules |
| **F — Security** | F1 | Access control rules |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `workmail:*`, `ses:Get*`, `route53:ListResourceRecordSets`, `ds:Describe*`, `iam:Get*` permissions

---

## License

MIT-0
