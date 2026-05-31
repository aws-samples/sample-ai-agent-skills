---
name: managed-flink-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Managed Service
  for Apache Flink (formerly Kinesis Data Analytics) problems by
  analyzing application creation, startup, Flink job failures,
  checkpointing, source/sink connectors, scaling, parallelism, state
  management, watermarks, VPC connectivity, IAM permissions, Studio
  notebooks, and following structured runbooks.
  Activate when: application creation failures, start errors, Flink
  job failures, checkpoint issues, connector errors, scaling problems,
  state management issues, watermark problems, VPC connectivity
  failures, IAM errors, Studio notebook issues, or the user says
  something is wrong with Managed Flink or Kinesis Data Analytics.
compatibility: >
  Requires AWS CLI or SDK access with kinesisanalyticsv2, kinesis,
  s3, iam, cloudwatch, logs, ec2, and cloudtrail permissions.
---

# Amazon Managed Service for Apache Flink Diagnostics

## When to use

Any Managed Flink investigation — application creation, startup, Flink job failures, checkpointing, source/sink connectors, scaling, parallelism, state management, watermarks, VPC connectivity, IAM permissions, or Studio notebooks.

## Investigation workflow

### Step 1 — Collect and triage

```
aws kinesisanalyticsv2 list-applications --query 'ApplicationSummaries[*].{Name:ApplicationName,Status:ApplicationStatus,RuntimeEnvironment:RuntimeEnvironment}'
aws kinesisanalyticsv2 describe-application --application-name <app-name>
aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name downtime --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws kinesisanalyticsv2 describe-application --application-name <app-name> --include-additional-details
aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --start-time <epoch-ms> --limit 100
aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name lastCheckpointDuration --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Average
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kinesisanalytics.amazonaws.com --max-results 20
aws kinesisanalyticsv2 describe-application-snapshot --application-name <app-name> --snapshot-name <snapshot>
aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name numberOfFailedCheckpoints --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any Managed Flink issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `kinesisanalyticsv2 list-applications` | List all Flink applications |
| `kinesisanalyticsv2 describe-application` | Get application details |
| `kinesisanalyticsv2 start-application` | Start a stopped application |
| `kinesisanalyticsv2 stop-application` | Stop a running application |
| `kinesisanalyticsv2 update-application` | Update application configuration |
| `kinesisanalyticsv2 create-application-snapshot` | Create state snapshot |
| `kinesisanalyticsv2 describe-application-snapshot` | Check snapshot status |

## Gotchas: Amazon Managed Service for Apache Flink

- The service was renamed from Kinesis Data Analytics to Managed Service for Apache Flink. The API namespace is still `kinesisanalyticsv2`.
- Applications have TWO runtime environments: FLINK-1_15, FLINK-1_18, etc. for code-based, and ZEPPELIN-FLINK-3_0 for Studio notebooks.
- Checkpointing is critical for fault tolerance. Failed checkpoints mean the application cannot recover from failures without data loss.
- Parallelism and ParallelismPerKPU control scaling. Total parallelism = Parallelism × ParallelismPerKPU. Incorrect values cause resource waste or bottlenecks.
- VPC-connected applications cannot access the internet by default. They need NAT Gateway for internet access and VPC endpoints for AWS services.
- Application snapshots capture Flink state. They are required for safe updates and rollbacks. Always snapshot before making changes.
- Studio notebooks (Zeppelin) have different resource limits and capabilities than code-based applications.

## Anti-hallucination rules

1. Always cite specific application names, CloudWatch metrics, or log entries as evidence.
2. Kinesis Data Analytics v2 API and Managed Flink are the same service. Never treat them as separate.
3. Checkpointing and savepointing are different mechanisms. Never conflate them.
4. Parallelism and ParallelismPerKPU multiply together. Never confuse individual settings with total parallelism.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Application | A1-A2 | Application creation failures, application start errors |
| B — Flink Jobs | B1-B2 | Flink job failures, checkpoint failures |
| C — Connectors | C1-C2 | Source connector errors, sink connector errors |
| D — Scaling | D1-D2 | Scaling issues, parallelism configuration |
| E — State & Time | E1-E2 | State management, watermark issues |
| F — Infrastructure | F1-F2 | VPC connectivity, IAM permissions |
| G — Studio | G1 | Studio notebook issues |
| Z — Catch-All | Z1 | General troubleshooting |
