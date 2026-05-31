---
name: personalize-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Personalize
  problems by analyzing dataset import, solution training, campaign
  deployment, real-time recommendations, batch inference, event
  tracking, filters, domain recommenders, and following structured
  runbooks. Activate when: dataset import failures, solution training
  errors, campaign deployment issues, poor recommendation quality,
  batch inference failures, event tracking problems, filter issues,
  or the user says something is wrong with Personalize.
compatibility: >
  Requires AWS CLI or SDK access with Personalize, Personalize Runtime,
  Personalize Events, S3, IAM, CloudWatch, and CloudTrail permissions.
---

# Amazon Personalize Diagnostics

## When to use

Any Amazon Personalize investigation where the console alone is insufficient — dataset management, solution training, campaign deployment, recommendation quality, batch inference, event tracking, or filter configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws personalize list-dataset-groups
aws personalize list-solutions
aws personalize list-campaigns
aws personalize list-batch-inference-jobs
```

### Step 2 — Domain deep dive

```
aws personalize describe-dataset-group --dataset-group-arn <arn>
aws personalize describe-solution --solution-arn <arn>
aws personalize describe-solution-version --solution-version-arn <arn>
aws personalize describe-campaign --campaign-arn <arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=personalize.amazonaws.com --max-results 20
aws personalize list-dataset-import-jobs --dataset-arn <arn>
aws personalize describe-dataset-import-job --dataset-import-job-arn <arn>
```

Read `references/guardrails.md` before concluding on any Personalize issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `personalize list-dataset-groups` | List dataset groups |
| `personalize describe-solution-version` | Check training status |
| `personalize describe-campaign` | Check campaign status |
| `personalize-runtime get-recommendations` | Get real-time recommendations |
| `personalize-events put-events` | Send real-time events |
| `personalize list-batch-inference-jobs` | List batch jobs |
| `personalize list-filters` | List recommendation filters |

## Gotchas: Amazon Personalize

- Dataset import requires specific CSV schema with required columns (USER_ID, ITEM_ID, TIMESTAMP for interactions).
- Solution training can take hours. Training creates a solution version. Multiple recipes available for different use cases.
- Campaigns must be created from a solution version for real-time recommendations. Campaigns are billed while active.
- Real-time events via PutEvents take up to 15 minutes to influence recommendations.
- Batch inference requires S3 input/output. Input format is JSON lines with userId or itemId.
- Filters use expressions to include/exclude items. Filter expressions have specific syntax.
- Cold start (new users/items) requires specific recipes (USER_PERSONALIZATION with exploration).
- Minimum data requirements: 1000 interactions, 25 unique users, 2 interactions per user.

## Anti-hallucination rules

1. Always cite specific ARNs, job IDs, or API responses as evidence.
2. Training takes hours. Never assume immediate model availability.
3. Campaigns are billed while active. Never assume free inference.
4. Real-time events take up to 15 minutes. Never assume instant influence.
5. Minimum data requirements are strict. Never assume any data volume works.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Datasets | A1-A3 | Dataset import, schema, data quality |
| B — Solutions | B1-B2 | Solution training, recipe selection |
| C — Campaigns | C1-C2 | Campaign deployment, campaign updates |
| D — Recommendations | D1-D2 | Real-time recommendations, quality |
| E — Batch | E1-E2 | Batch inference, batch segment |
| F — Events | F1-F2 | Event tracking, event ingestion |
| G — Filters | G1 | Filter configuration |
| H — Domain | H1 | Domain recommenders |
| Z — Catch-All | Z1 | General troubleshooting |
