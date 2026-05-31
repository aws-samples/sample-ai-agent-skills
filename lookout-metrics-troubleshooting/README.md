# Amazon Lookout for Metrics Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Lookout for Metrics problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Lookout for Metrics — detector creation, activation, dataset configuration, data source connectivity, anomaly detection tuning, false positive management, alerting, and data source integrations.

### Activate When

- Detector creation failures
- Detector activation errors
- Dataset configuration issues
- Data source connectivity problems
- Anomaly detection not working
- False positive anomalies
- Alert configuration errors
- Lambda alert invocation failures
- S3 data source ingestion issues
- CloudWatch data source problems
- Redshift/RDS data source errors
- General Lookout for Metrics errors

---

## Skill Structure

```
lookout-metrics-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-detector-creation-failures.md
    ├── A2-detector-activation-errors.md
    ├── B1-dataset-configuration-errors.md
    ├── B2-data-source-connectivity.md
    ├── C1-anomaly-detection-issues.md
    ├── C2-false-positive-management.md
    ├── D1-alert-configuration.md
    ├── D2-lambda-alert-errors.md
    ├── E1-s3-data-source.md
    ├── E2-cloudwatch-data-source.md
    ├── F1-redshift-rds-data-source.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Detector** | A1–A2 | Creation failures, activation errors |
| **B — Dataset** | B1–B2 | Configuration errors, data source connectivity |
| **C — Detection** | C1–C2 | Anomaly detection issues, false positive management |
| **D — Alerting** | D1–D2 | Alert configuration, Lambda alert errors |
| **E — S3/CloudWatch** | E1–E2 | S3 data source, CloudWatch data source |
| **F — Database Sources** | F1 | Redshift/RDS data source |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `lookoutmetrics:*`, `s3:Get*`, `cloudwatch:GetMetricData`, `lambda:GetPolicy`, `redshift:DescribeClusters`, `rds:DescribeDBInstances`, `iam:Get*` permissions

---

## License

MIT-0
