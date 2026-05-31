# Amazon Macie Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Macie problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Macie when the console alone isn't enough — discovery job failures, classification accuracy, custom data identifiers, finding analysis, S3 bucket inventory, multi-account management, cost optimization, and integration configuration.

### Activate When

- Discovery jobs failing or stuck
- Classification results inaccurate
- Custom data identifiers not matching expected patterns
- Unexpected or missing findings
- False positive management
- S3 bucket inventory incomplete
- Sensitive data not detected in known locations
- Delegated admin configuration issues
- Member account enrollment problems
- High Macie costs
- Sampling configuration questions
- Security Hub findings not appearing
- EventBridge rules not triggering
- General Macie errors without clear symptoms

---

## Skill Structure

```
macie-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-discovery-job-failures.md
    ├── A2-classification-errors.md
    ├── A3-custom-data-identifiers.md
    ├── B1-finding-analysis.md
    ├── B2-false-positives.md
    ├── C1-bucket-inventory.md
    ├── C2-sensitive-data-discovery.md
    ├── D1-delegated-admin.md
    ├── D2-member-accounts.md
    ├── E1-cost-optimization.md
    ├── E2-sampling.md
    ├── F1-security-hub.md
    ├── F2-eventbridge.md
    ├── Z1-general-troubleshooting.md
    ├── macie-guardrails.md
    └── macie-hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Discovery** | A1–A3 | Discovery job failures, classification errors, custom data identifiers |
| **B — Findings** | B1–B2 | Finding analysis, false positives |
| **C — S3** | C1–C2 | Bucket inventory, sensitive data discovery |
| **D — Multi-Account** | D1–D2 | Delegated admin, member accounts |
| **E — Cost** | E1–E2 | Cost optimization, sampling |
| **F — Integration** | F1–F2 | Security Hub, EventBridge |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/macie-guardrails.md` covering S3-only scanning scope, discovery job scheduling, custom identifier regex limits, managed vs custom identifiers, regional findings, bucket inventory vs scanning, sampling trade-offs, per-region enablement, suppression behavior, pricing model, classification result types, and allow-list configuration.

---

## Investigation Workflow

1. **Triage** — Check Macie session, job status, findings summary, bucket inventory
2. **Deep Dive** — Examine job details, custom identifiers, finding statistics, sensitive data occurrences
3. **Detailed** — Review multi-account setup, usage totals, CloudTrail events

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `macie2:*`, `s3:ListAllMyBuckets`, `s3:GetBucketLocation`, `securityhub:GetFindings`, `events:ListRules`
- Macie enabled in the account/region
- S3 bucket access for discovery jobs

---

## License

MIT-0
