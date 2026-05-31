---
name: forecast-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Forecast
  problems by analyzing dataset import, predictor training, forecast
  generation, accuracy metrics, what-if analysis, explainability,
  auto-predictor, and following structured runbooks. Activate when:
  dataset import failures, predictor training errors, forecast
  generation issues, poor accuracy, what-if analysis problems,
  explainability errors, or the user says something is wrong with
  Forecast without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Forecast, Forecast Query, S3,
  IAM, CloudWatch, and CloudTrail permissions.
---

# Amazon Forecast Diagnostics

## When to use

Any Amazon Forecast investigation where the console alone is insufficient — dataset management, predictor training, forecast generation, accuracy evaluation, what-if analysis, or explainability.

## Investigation workflow

### Step 1 — Collect and triage

```
aws forecast list-dataset-groups
aws forecast list-predictors
aws forecast list-forecasts
aws forecast list-what-if-analyses
```

### Step 2 — Domain deep dive

```
aws forecast describe-dataset-group --dataset-group-arn <arn>
aws forecast describe-auto-predictor --predictor-arn <arn>
aws forecast describe-forecast --forecast-arn <arn>
aws forecast get-accuracy-metrics --predictor-arn <arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=forecast.amazonaws.com --max-results 20
aws forecast describe-dataset-import-job --dataset-import-job-arn <arn>
```

Read `references/guardrails.md` before concluding on any Forecast issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `forecast list-dataset-groups` | List dataset groups |
| `forecast describe-auto-predictor` | Check predictor status |
| `forecast describe-forecast` | Check forecast status |
| `forecast get-accuracy-metrics` | Get predictor accuracy |
| `forecastquery query-forecast` | Query forecast values |
| `forecast list-what-if-analyses` | List what-if analyses |
| `forecast list-explainabilities` | List explainability jobs |

## Gotchas: Amazon Forecast

- Dataset import requires specific CSV format with timestamp, target value, and item ID columns.
- AutoPredictor is recommended over legacy predictors. It automatically selects best algorithm.
- Predictor training can take hours. Training creates an immutable predictor.
- Forecast generation is separate from training. Must create forecast from predictor.
- Accuracy metrics (WAPE, RMSE, MASE) are available after training. Lower WAPE is better.
- What-if analysis requires an existing forecast. It models scenarios with modified data.
- Explainability shows feature importance. Requires predictor with explainability enabled.
- Forecast horizon must match predictor's forecast horizon setting.

## Anti-hallucination rules

1. Always cite specific ARNs, job IDs, or API responses as evidence.
2. Training takes hours. Never assume immediate predictor availability.
3. Forecast generation is separate from training. Never assume automatic forecast.
4. AutoPredictor is recommended. Never default to legacy predictor APIs.
5. Accuracy metrics are predictor-specific. Never assume universal accuracy.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Datasets | A1-A2 | Dataset import, data format |
| B — Predictors | B1-B3 | Predictor training, AutoPredictor, accuracy |
| C — Forecasts | C1-C2 | Forecast generation, forecast query |
| D — What-If | D1-D2 | What-if analysis, what-if forecast |
| E — Explainability | E1 | Explainability analysis |
| F — Data Quality | F1-F2 | Time series quality, missing values |
| Z — Catch-All | Z1 | General troubleshooting |
