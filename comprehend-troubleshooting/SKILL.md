---
name: comprehend-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Comprehend
  problems by analyzing entity recognition, sentiment analysis, custom
  classifiers, custom entity recognition, async jobs, PII detection,
  key phrase extraction, language detection, topic modeling, and
  following structured runbooks. Activate when: entity recognition
  errors, sentiment analysis issues, custom classifier training
  failures, custom entity model problems, async job failures, PII
  detection inaccuracies, or the user says something is wrong with
  Comprehend without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Comprehend, S3, IAM, CloudWatch,
  and CloudTrail permissions.
---

# Amazon Comprehend Diagnostics

## When to use

Any Amazon Comprehend investigation where the console alone is insufficient — entity recognition, sentiment analysis, custom classifiers, custom entity recognition, async batch processing, PII detection, or topic modeling.

## Investigation workflow

### Step 1 — Collect and triage

```
aws comprehend list-endpoints
aws comprehend list-entity-recognizers
aws comprehend list-document-classifiers
aws comprehend list-pii-entities-detection-jobs
```

### Step 2 — Domain deep dive

```
aws comprehend describe-entity-recognizer --entity-recognizer-arn <arn>
aws comprehend describe-document-classifier --document-classifier-arn <arn>
aws comprehend detect-sentiment --text "sample text" --language-code en
aws comprehend detect-entities --text "sample text" --language-code en
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=comprehend.amazonaws.com --max-results 20
aws comprehend describe-dominant-language-detection-job --job-id <job-id>
```

Read `references/guardrails.md` before concluding on any Comprehend issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `comprehend detect-entities` | Real-time entity detection |
| `comprehend detect-sentiment` | Real-time sentiment analysis |
| `comprehend detect-pii-entities` | Real-time PII detection |
| `comprehend detect-key-phrases` | Real-time key phrase extraction |
| `comprehend detect-dominant-language` | Language detection |
| `comprehend list-document-classifiers` | List custom classifiers |
| `comprehend list-entity-recognizers` | List custom entity models |
| `comprehend list-endpoints` | List inference endpoints |

## Gotchas: Amazon Comprehend

- Real-time APIs have text size limits (100KB for most). Use async batch jobs for large volumes.
- Custom classifiers require minimum training data (varies by mode: multi-class vs multi-label).
- Custom entity recognizers need annotated training data in specific format.
- Endpoints must be created for custom model inference. Endpoints are billed while active.
- PII detection supports specific entity types. Not all PII types are detected in all languages.
- Async jobs require S3 input/output locations with proper IAM permissions.
- Language detection should be run first if language is unknown. Most APIs require language code.
- Topic modeling requires minimum 1000 documents for meaningful results.

## Anti-hallucination rules

1. Always cite specific model ARNs, job IDs, or API responses as evidence.
2. Custom models require endpoints for inference. Never assume direct model invocation.
3. Real-time APIs have size limits. Never assume unlimited text input.
4. PII detection varies by language. Never assume all PII types in all languages.
5. Async jobs require S3. Never assume direct text input for batch processing.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Entity Recognition | A1-A2 | Built-in entities, custom entities |
| B — Sentiment | B1-B2 | Sentiment analysis, targeted sentiment |
| C — Custom Classifiers | C1-C3 | Training, deployment, inference |
| D — PII Detection | D1-D2 | PII detection, PII redaction |
| E — Async Jobs | E1-E2 | Batch processing, job management |
| F — Language | F1 | Language detection |
| Z — Catch-All | Z1 | General troubleshooting |
