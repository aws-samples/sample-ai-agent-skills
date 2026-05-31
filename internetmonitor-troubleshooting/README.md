# Amazon CloudWatch Internet Monitor Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon CloudWatch Internet Monitor problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for CloudWatch Internet Monitor — monitor creation, configuration, health events, availability scores, traffic insights, city-network analysis, CloudFront/VPC integrations, log delivery, and threshold configuration.

### Activate When

- Monitor creation failures
- Monitor configuration issues
- Health event analysis problems
- Availability score anomalies
- Traffic insights errors
- City-network analysis issues
- CloudFront or VPC integration failures
- S3 or CloudWatch log delivery problems
- Threshold configuration issues
- General Internet Monitor errors

---

## Skill Structure

```
internetmonitor-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-monitor-creation-failures.md
    ├── A2-monitor-configuration.md
    ├── B1-health-event-analysis.md
    ├── B2-availability-score-issues.md
    ├── C1-traffic-insights-errors.md
    ├── C2-city-network-analysis.md
    ├── D1-cloudfront-integration.md
    ├── D2-vpc-integration.md
    ├── E1-s3-log-delivery.md
    ├── E2-cloudwatch-log-delivery.md
    ├── F1-threshold-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Monitor Setup** | A1–A2 | Creation failures, configuration |
| **B — Health Events** | B1–B2 | Health event analysis, availability score issues |
| **C — Traffic Insights** | C1–C2 | Traffic insights errors, city-network analysis |
| **D — Integrations** | D1–D2 | CloudFront integration, VPC integration |
| **E — Log Delivery** | E1–E2 | S3 log delivery, CloudWatch log delivery |
| **F — Thresholds** | F1 | Threshold configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `internetmonitor:*`, `cloudfront:Get*`, `ec2:DescribeVpcs`, `s3:GetBucketPolicy`, `logs:Describe*`, `cloudwatch:Get*` permissions

---

## License

MIT-0
