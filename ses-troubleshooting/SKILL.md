---
name: ses-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon SES problems
  by analyzing sending failures, deliverability, reputation, receiving
  rules, templates, configuration sets, sandbox limitations, security,
  and quota issues following structured runbooks. Activate when: send
  failures, bounce handling, complaint handling, reputation issues,
  DKIM/SPF/DMARC failures, suppression list problems, receipt rule
  errors, S3 action failures, template errors, personalization issues,
  configuration set problems, event destination failures, sandbox
  limitations, production access requests, IAM permission errors,
  SMTP credential issues, sending limits, rate throttling, or the
  user says something is wrong with SES without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SES v2, IAM, S3, SNS, CloudWatch,
  and optionally Route 53 permissions.
---

# SES Diagnostics

## When to use

Any SES investigation where the console alone is insufficient — send failures, bounces, complaints, deliverability issues, authentication failures, receiving configuration, template problems, sandbox restrictions, or quota management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws sesv2 get-account
aws sesv2 get-email-identity --email-identity <domain-or-email>
aws sesv2 get-suppression-list-attributes
aws sesv2 list-suppression-destinations
aws cloudwatch get-metric-statistics --namespace AWS/SES --metric-name Reputation.BounceRate --start-time <iso> --end-time <iso> --period 86400 --statistics Average
```

### Step 2 — Domain deep dive

```
aws sesv2 get-deliverability-dashboard-options
aws sesv2 list-configuration-sets
aws sesv2 get-configuration-set --configuration-set-name <name>
aws ses describe-active-receipt-rule-set
aws ses describe-receipt-rule --rule-set-name <name> --rule-name <rule>
aws sesv2 get-account --query 'SendQuota'
```

Read `references/guardrails.md` before concluding on any SES issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `sesv2 get-account` | Account status, sandbox mode, sending quota |
| `sesv2 get-email-identity` | Identity verification, DKIM, MAIL FROM |
| `sesv2 get-suppression-list-attributes` | Account-level suppression list config |
| `sesv2 list-suppressed-destinations` | Suppressed email addresses |
| `ses describe-active-receipt-rule-set` | Active receiving rule set |
| `ses describe-receipt-rule` | Individual receipt rule configuration |
| `sesv2 list-configuration-sets` | Configuration sets for tracking |
| `sesv2 get-configuration-set` | Configuration set details and event destinations |
| `cloudwatch get-metric-statistics` | SES reputation and sending metrics |

## Gotchas: SES

- SES sandbox mode limits sending to verified addresses only. Production access must be requested and approved by AWS.
- Bounce rate above 5% or complaint rate above 0.1% triggers account review and potential sending pause.
- DKIM signing requires three CNAME records in DNS. All three must be present for DKIM to pass.
- SPF alignment requires a custom MAIL FROM domain. Without it, SPF passes for amazonses.com, not your domain.
- The account-level suppression list automatically suppresses addresses that bounce or complain. Manually removing addresses does not prevent re-addition.
- Receipt rules process in order. The first matching rule handles the email. Subsequent rules are not evaluated.
- SES has a per-second sending rate limit separate from the daily sending quota. Both must be respected.
- Configuration set event destinations can publish to CloudWatch, SNS, Kinesis Firehose, or Pinpoint. Each has different latency characteristics.
- SMTP credentials are derived from IAM user access keys using a specific algorithm. They are not the same as IAM credentials.
- SES sending from Lambda requires the Lambda execution role to have ses:SendEmail permission, not the Lambda function's resource policy.

## Anti-hallucination rules

1. Always cite specific account status, identity verification state, bounce/complaint rates, or sending quota as evidence.
2. Sandbox mode is not a bug. Never claim SES is broken when the account is in sandbox mode.
3. SMTP credentials are NOT IAM access keys. Never suggest using IAM access key/secret directly for SMTP authentication.
4. Bounce rate threshold is 5%, complaint rate threshold is 0.1%. Never cite different thresholds.
5. Receipt rules are ordered. Never claim all matching rules execute.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Sending | A1–A3 | Send failures, bounce handling, complaint handling |
| B — Deliverability | B1–B3 | Reputation, DKIM/SPF/DMARC, suppression list |
| C — Receiving | C1–C2 | Receipt rules, S3 actions |
| D — Templates | D1–D2 | Template errors, personalization |
| E — Configuration | E1–E2 | Configuration sets, event destinations |
| F — Sandbox | F1–F2 | Sandbox limitations, production access |
| G — Security | G1–G2 | IAM permissions, SMTP credentials |
| H — Quotas | H1–H2 | Sending limits, rate throttling |
| Z — Catch-All | Z1 | General troubleshooting |
