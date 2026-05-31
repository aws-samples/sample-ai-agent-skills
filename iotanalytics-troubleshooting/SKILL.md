---
name: iotanalytics-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT Analytics problems
  by analyzing channels, pipelines, data stores, datasets, SQL queries,
  container activities, and data retention using structured runbooks.
  Activate when: channel ingestion failures, pipeline processing errors,
  data store query issues, dataset computation failures, SQL errors,
  container activity problems, or the user says something is wrong with
  IoT Analytics.
compatibility: >
  Requires AWS CLI or SDK access with IoTAnalytics, IAM, CloudWatch,
  S3, and Lambda permissions as applicable.
---

# AWS IoT Analytics Diagnostics

## When to use

Any AWS IoT Analytics investigation — channels, pipelines, data stores, datasets, SQL queries, container activities, or data retention troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iotanalytics list-channels
aws iotanalytics describe-channel --channel-name <name>
aws iotanalytics list-pipelines
aws iotanalytics describe-pipeline --pipeline-name <name>
```

### Step 2 — Domain deep dive

```
aws iotanalytics list-datastores
aws iotanalytics describe-datastore --datastore-name <name>
aws iotanalytics list-datasets
aws iotanalytics describe-dataset --dataset-name <name>
aws iotanalytics get-dataset-content --dataset-name <name>
```

### Step 3 — Detailed investigation

```
aws iotanalytics sample-channel-data --channel-name <name>
aws iotanalytics run-pipeline-activity --pipeline-activities '[<activity>]' --payloads '<base64>'
aws cloudwatch get-metric-statistics --namespace AWS/IoTAnalytics --metric-name IncomingMessages --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any IoT Analytics issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-channel` | Check channel configuration and status |
| `describe-pipeline` | Check pipeline activities and errors |
| `describe-datastore` | Check data store configuration |
| `describe-dataset` | Check dataset queries and schedule |
| `sample-channel-data` | Get sample messages from channel |
| `run-pipeline-activity` | Test pipeline activity with sample data |
| CloudWatch Metrics | Check ingestion and processing rates |

## Anti-hallucination rules

1. Always cite specific channel, pipeline, or dataset names as evidence.
2. Pipelines process data sequentially through activities. Never assume parallel processing.
3. Datasets are computed results, not raw data stores. Never confuse them.
4. SQL datasets use a specific SQL dialect. Never assume standard SQL compatibility.
5. Container datasets run custom Docker containers. Never assume Lambda execution.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Channels | A1-A2 | Ingestion failures, channel configuration |
| B — Pipelines | B1-B3 | Processing errors, activity failures, transforms |
| C — Data Stores | C1-C2 | Storage issues, partitioning |
| D — Datasets | D1-D3 | SQL queries, container activities, scheduling |
| E — Retention | E1-E2 | Data retention, cleanup |
| Z — Catch-All | Z1 | General troubleshooting |
