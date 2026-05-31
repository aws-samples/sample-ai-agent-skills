---
name: workmail-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon WorkMail
  problems by analyzing organization creation, domain verification,
  email delivery, receiving issues, client connectivity, mobile sync,
  mailbox issues, group management, migration, journaling, and access
  control rules.
  Activate when: organization creation failures, domain verification
  issues, email delivery failures, client connectivity problems,
  mailbox errors, migration issues, or the user says something is
  wrong with WorkMail.
compatibility: >
  Requires AWS CLI or SDK access with workmail, ses, route53,
  ds (Directory Service), iam, cloudtrail, and cloudwatch permissions.
---

# Amazon WorkMail Diagnostics

## When to use

Any Amazon WorkMail investigation — organization creation, domain verification, email delivery failures, receiving issues, client connectivity, mobile device sync, mailbox issues, group management, migration from Exchange, journaling rules, or access control rules.

## Investigation workflow

### Step 1 — Collect and triage

```
aws workmail list-organizations --query 'OrganizationSummaries[*].{Id:OrganizationId,Alias:Alias,State:State}'
aws workmail list-users --organization-id <org-id> --query 'Users[*].{Id:EntityId,Name:Name,State:State,Role:UserRole}'
aws workmail list-mailbox-export-jobs --organization-id <org-id>
```

### Step 2 — Domain deep dive

```
aws workmail describe-organization --organization-id <org-id>
aws workmail list-mail-domains --organization-id <org-id>
aws workmail get-mail-domain --organization-id <org-id> --domain-name <domain>
```

### Step 3 — Detailed investigation

```
aws workmail describe-user --organization-id <org-id> --user-id <user-id>
aws workmail list-access-control-rules --organization-id <org-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workmail.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any WorkMail issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `workmail list-organizations` | List WorkMail organizations |
| `workmail describe-organization` | Get organization details |
| `workmail list-users` | List users and their state |
| `workmail describe-user` | Get user mailbox details |
| `workmail list-mail-domains` | List configured domains |
| `workmail get-mail-domain` | Check domain verification status |
| `workmail list-access-control-rules` | List access control rules |

## Gotchas: Amazon WorkMail

- WorkMail organizations are REGIONAL. Each organization is tied to a specific region. Email routing, mailboxes, and configuration do not replicate across regions.
- WorkMail requires a directory (auto-created or existing). Deleting the directory destroys the organization and all mailboxes.
- Domain verification requires specific DNS records (MX, TXT, CNAME for DKIM). Incomplete DNS configuration causes delivery failures.
- WorkMail uses SES for outbound email. SES sending limits and reputation affect WorkMail delivery.
- Mobile device sync uses Exchange ActiveSync (EAS) protocol. Issues may be device-specific, not service-side.
- Access control rules can block specific IP ranges, protocols, or devices. Overly restrictive rules cause connectivity failures.
- Migration from Exchange requires the WorkMail Migration Tool. It does not support direct mailbox import from PST files.

## Anti-hallucination rules

1. Always cite specific organization IDs, user IDs, or API responses as evidence.
2. WorkMail is NOT SES. Never suggest SES APIs for mailbox management.
3. Domain verification requires DNS records. Never claim domains are auto-verified.
4. Mobile sync uses EAS protocol. Never suggest IMAP/POP for mobile device configuration.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Organization | A1–A2 | Organization creation, domain verification |
| B — Email Flow | B1–B2 | Delivery failures, receiving issues |
| C — Clients | C1–C2 | Client connectivity, mobile device sync |
| D — Mailboxes | D1–D2 | User mailbox issues, group management |
| E — Operations | E1–E2 | Migration from Exchange, journaling rules |
| F — Security | F1 | Access control rules |
| Z — Catch-All | Z1 | General troubleshooting |
