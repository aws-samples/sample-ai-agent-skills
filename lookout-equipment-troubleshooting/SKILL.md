---
name: lookout-equipment-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Lookout for Equipment
  problems by analyzing dataset creation, data ingestion, model training,
  inference scheduling, label management, retraining policies, S3 data access,
  IAM permissions, data quality, and following structured runbooks.
  Activate when: dataset creation failures, data ingestion errors, model
  training failures, inference scheduler issues, label problems, retraining
  policy errors, S3 access issues, IAM permission errors, data quality
  problems, or the user says something is wrong with Lookout for Equipment.
compatibility: >
  Requires AWS CLI or SDK access with lookoutequipment, s3, iam,
  cloudtrail, cloudwatch, and logs permissions.
---

# Amazon Lookout for Equipment Diagnostics

## When to use

Any Amazon Lookout for Equipment investigation — dataset creation, data ingestion, model training, model evaluation, inference scheduling, inference execution, label management, retraining policies, S3 data access, IAM permissions, or data quality issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws lookoutequipment list-datasets --query 'DatasetSummaries[*].{Name:DatasetName,Arn:DatasetArn,Status:Status,CreatedAt:CreatedAt}'
aws lookoutequipment list-models --query 'ModelSummaries[*].{Name:ModelName,DatasetName:DatasetName,Status:Status,CreatedAt:CreatedAt}'
aws lookoutequipment list-inference-schedulers --query 'InferenceSchedulerSummaries[*].{Name:InferenceSchedulerName,ModelName:ModelName,Status:Status}'
```

### Step 2 — Domain deep dive

```
aws lookoutequipment describe-dataset --dataset-name <name>
aws lookoutequipment describe-model --model-name <name>
aws lookoutequipment describe-inference-scheduler --inference-scheduler-name <name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutequipment.amazonaws.com --max-results 20
aws lookoutequipment list-inference-executions --inference-scheduler-name <name> --status FAILED
aws lookoutequipment describe-data-ingestion-job --job-id <job-id>
```

Read `references/guardrails.md` before concluding on any Lookout for Equipment issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `lookoutequipment list-datasets` | List all datasets |
| `lookoutequipment describe-dataset` | Get dataset details and schema |
| `lookoutequipment describe-data-ingestion-job` | Check ingestion job status |
| `lookoutequipment list-models` | List all models |
| `lookoutequipment describe-model` | Get model training details and metrics |
| `lookoutequipment list-inference-schedulers` | List inference schedulers |
| `lookoutequipment describe-inference-scheduler` | Get scheduler configuration |
| `lookoutequipment list-inference-executions` | List inference runs with status |
| `lookoutequipment list-labels` | List labels for a label group |
| `lookoutequipment describe-retraining-scheduler` | Get retraining policy details |

## Gotchas: Amazon Lookout for Equipment

- Lookout for Equipment requires sensor data in a specific CSV format with timestamps. Each component must be in a separate CSV file within a named subfolder in S3.
- The service requires a MINIMUM of 14 days of sensor data for training. Less data causes training failures. AWS recommends 90+ days for best results.
- Inference schedulers run on a fixed cadence (5 min, 10 min, 15 min, 30 min, 1 hr). You cannot trigger inference on-demand through the scheduler.
- Model training can take hours to days depending on dataset size. There is no way to speed up training or get intermediate results.
- Data timestamps must be in ISO 8601 format and must be monotonically increasing. Gaps or duplicates cause ingestion failures.
- The service is available in limited AWS regions. Check regional availability before troubleshooting connectivity issues.
- Retraining schedulers require the original training data to remain accessible in S3. Moving or deleting training data breaks retraining.

## Anti-hallucination rules

1. Always cite specific dataset names, model ARNs, job IDs, or API responses as evidence.
2. Lookout for Equipment requires structured CSV sensor data — never suggest it works with unstructured data or images.
3. Minimum 14 days of training data is required. Never suggest training with less data will succeed.
4. Inference schedulers run on fixed intervals — never suggest on-demand inference through the scheduler API.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Dataset | A1-A2 | Dataset creation failures, data ingestion errors |
| B — Model | B1-B2 | Model training failures, model evaluation issues |
| C — Inference | C1-C2 | Inference scheduler errors, inference execution failures |
| D — Labels & Retraining | D1-D2 | Label creation issues, retraining policy |
| E — Access | E1-E2 | S3 data access, IAM permissions |
| F — Data Quality | F1 | Data quality issues |
| Z — Catch-All | Z1 | General troubleshooting |
