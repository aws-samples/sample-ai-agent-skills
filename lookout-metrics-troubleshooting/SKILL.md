---
name: lookout-metrics-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Lookout for Metrics
  problems by analyzing detector creation, dataset configuration, data source
  connectivity, anomaly detection, alerting, and following structured runbooks.
  Activate when: detector creation failures, activation errors, dataset issues,
  data source connectivity problems, anomaly detection failures, false positives,
  alert configuration errors, or the user says something is wrong with Lookout for Metrics.
compatibility: >
  Requires AWS CLI or SDK access with lookoutmetrics, s3, cloudwatch, lambda,
  redshift, rds, iam, and cloudtrail permissions.
---

# Amazon Lookout for Metrics Diagnostics

## When to use

Any Amazon Lookout for Metrics investigation — detector creation, activation, dataset configuration, data source connectivity, anomaly detection tuning, false positive management, alert configuration, or data source integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws lookoutmetrics list-anomaly-detectors --query 'AnomalyDetectorSummaryList[*].{Name:AnomalyDetectorName,Arn:AnomalyDetectorArn,Status:Status}'
aws lookoutmetrics list-anomaly-group-summaries --anomaly-detector-arn <arn> --sensitivity-threshold 50
aws lookoutmetrics list-metric-sets --anomaly-detector-arn <arn>
```

### Step 2 — Domain deep dive

```
aws lookoutmetrics describe-anomaly-detector --anomaly-detector-arn <arn>
aws lookoutmetrics describe-metric-set --metric-set-arn <arn>
aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutmetrics.amazonaws.com --max-results 20
aws lookoutmetrics list-alerts --anomaly-detector-arn <arn>
aws lookoutmetrics describe-alert --alert-arn <arn>
```

Read `references/guardrails.md` before concluding on any Lookout for Metrics issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `lookoutmetrics list-anomaly-detectors` | List all detectors and their status |
| `lookoutmetrics describe-anomaly-detector` | Get detector configuration details |
| `lookoutmetrics list-metric-sets` | List metric sets for a detector |
| `lookoutmetrics describe-metric-set` | Get metric set configuration |
| `lookoutmetrics get-data-quality-metrics` | Check data quality and completeness |
| `lookoutmetrics list-anomaly-group-summaries` | List detected anomaly groups |
| `lookoutmetrics list-alerts` | List configured alerts |

## Gotchas: Amazon Lookout for Metrics

- Detectors require a learning period (cold start) of 1–2 weeks before producing reliable anomaly results. Do not expect immediate detection.
- Metric sets define the measures, dimensions, and timestamps. A misconfigured timestamp format is the most common cause of dataset errors.
- S3 data sources require a specific folder structure with date-partitioned paths. Incorrect path templates cause silent data ingestion failures.
- CloudWatch data sources require the metric namespace, name, and dimensions to match exactly. Partial matches return no data.
- Anomaly sensitivity threshold (1–100) controls detection aggressiveness. Lower values detect fewer anomalies with higher confidence; higher values detect more but increase false positives.
- Alerts can target SNS topics or Lambda functions. Lambda alerts require the function to have a resource-based policy granting Lookout for Metrics invoke permission.
- Detectors cannot be modified while in ACTIVE status. You must deactivate, modify, then reactivate — which resets the learning period.

## Anti-hallucination rules

1. Always cite specific detector ARNs, metric set ARNs, or API responses as evidence.
2. Lookout for Metrics has a mandatory learning period. Never claim instant anomaly detection.
3. Data quality metrics are the primary diagnostic tool. Always check them before blaming detection logic.
4. S3 path templates and timestamp formats are the top two root causes. Verify both before escalating.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Detector | A1–A2 | Creation failures, activation errors |
| B — Dataset | B1–B2 | Configuration errors, data source connectivity |
| C — Detection | C1–C2 | Anomaly detection issues, false positive management |
| D — Alerting | D1–D2 | Alert configuration, Lambda alert errors |
| E — S3/CloudWatch | E1–E2 | S3 data source, CloudWatch data source |
| F — Database Sources | F1 | Redshift/RDS data source |
| Z — Catch-All | Z1 | General troubleshooting |
