# AWS CloudTrail Diagnostics Skill

Agent skill for investigating and troubleshooting AWS CloudTrail problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS CloudTrail when the console alone isn't enough — trail configuration, log delivery, missing events, data events, organization trails, CloudTrail Lake, Insights, CloudWatch Logs integration, log validation, encryption, and cost management.

### Activate When

- Trail not delivering logs to S3
- Events missing from CloudTrail
- Data events not being captured
- Organization trail configuration issues
- CloudTrail Lake query failures
- Insights not detecting anomalies
- CloudWatch Logs not receiving events
- Log file validation failures
- KMS encryption errors
- Unexpected CloudTrail costs
- General CloudTrail errors without clear symptoms

---

## Skill Structure

```
cloudtrail-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-trail-creation.md
    ├── A2-delivery-failures.md
    ├── A3-s3-bucket-issues.md
    ├── B1-missing-events.md
    ├── B2-data-events.md
    ├── B3-management-events.md
    ├── C1-organization-trail.md
    ├── C2-delegated-admin.md
    ├── D1-lake-queries.md
    ├── D2-event-data-store.md
    ├── E1-insights-events.md
    ├── E2-anomaly-detection.md
    ├── F1-cloudwatch-logs.md
    ├── F2-eventbridge.md
    ├── G1-log-file-validation.md
    ├── G2-encryption.md
    ├── H1-data-event-costs.md
    ├── H2-lake-pricing.md
    ├── Z1-general-troubleshooting.md
    ├── cloudtrail-guardrails.md
    └── cloudtrail-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Trail** | A1–A3 | Trail creation, delivery failures, S3 bucket issues |
| **B — Events** | B1–B3 | Missing events, data events, management events |
| **C — Organization** | C1–C2 | Organization trail, delegated admin |
| **D — Lake** | D1–D2 | CloudTrail Lake queries, event data store |
| **E — Insights** | E1–E2 | Insights events, anomaly detection |
| **F — Integration** | F1–F2 | CloudWatch Logs, EventBridge |
| **G — Security** | G1–G2 | Log file validation, encryption |
| **H — Cost** | H1–H2 | Data event costs, Lake pricing |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/cloudtrail-guardrails.md` covering management vs data event distinction, log delivery latency, organization trail management, CloudTrail Lake vs S3 trails, log file validation, S3 bucket policy requirements, KMS key policy requirements, data event cost implications, multi-region trail behavior, event selector types, Insights prerequisites, and event history retention.

---

## Investigation Workflow

1. **Triage** — Check trail status, event selectors, recent events, delivery errors
2. **Deep Dive** — Examine Insights, Lake data stores, S3 bucket policy, KMS configuration
3. **Detailed** — Review Lake queries, CloudWatch Logs, log validation, cost analysis

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `cloudtrail:*`, `s3:GetBucketPolicy`, `logs:DescribeLogGroups`, `kms:DescribeKey`, `events:ListRules`
- At least one trail configured in the account

---

## License

MIT-0
