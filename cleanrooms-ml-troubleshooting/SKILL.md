---
name: cleanrooms-ml-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Clean Rooms ML
  problems by analyzing training model failures, configured model errors,
  audience generation issues, lookalike models, collaboration ML config,
  privacy budgets, seed audiences, output configuration, S3 data access,
  IAM permissions, differential privacy, and following structured runbooks.
  Activate when: training model failures, configured model errors,
  audience generation issues, lookalike model problems, collaboration
  ML configuration errors, privacy budget exhaustion, seed audience
  errors, output configuration failures, S3 data access issues, IAM
  permission errors, differential privacy problems, or the user says
  something is wrong with Clean Rooms ML.
compatibility: >
  Requires AWS CLI or SDK access with cleanrooms-ml, cleanrooms, s3,
  iam, kms, cloudtrail, and cloudwatch permissions.
---

# AWS Clean Rooms ML Diagnostics

## When to use

Any AWS Clean Rooms ML investigation — training model failures, configured model errors, audience generation issues, lookalike models, collaboration ML configuration, privacy budgets, seed audiences, output configuration, S3 data access, IAM permissions, or differential privacy.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cleanroomsml list-training-datasets --query 'trainingDatasets[*].{Name:name,Arn:trainingDatasetArn,Status:status,CreateTime:createTime}'
aws cleanroomsml list-configured-model-algorithms --query 'configuredModelAlgorithms[*].{Name:name,Arn:configuredModelAlgorithmArn,CreateTime:createTime}'
aws cleanroomsml list-audience-models --query 'audienceModels[*].{Name:name,Arn:audienceModelArn,Status:status}'
```

### Step 2 — Domain deep dive

```
aws cleanroomsml get-training-dataset --training-dataset-arn <arn>
aws cleanroomsml get-audience-model --audience-model-arn <arn>
aws cleanroomsml get-configured-audience-model --configured-audience-model-arn <arn>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cleanroomsml.amazonaws.com --max-results 20
aws cleanroomsml list-audience-generation-jobs --query 'audienceGenerationJobs[*].{Name:name,Arn:audienceGenerationJobArn,Status:status}'
aws cleanrooms list-collaborations --query 'collaborationList[*].{Name:name,Id:id,Status:status}'
```

Read `references/guardrails.md` before concluding on any Clean Rooms ML issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `cleanroomsml list-training-datasets` | List training datasets |
| `cleanroomsml get-training-dataset` | Get training dataset details |
| `cleanroomsml list-audience-models` | List audience models |
| `cleanroomsml get-audience-model` | Get audience model details |
| `cleanroomsml get-configured-audience-model` | Get configured audience model |
| `cleanroomsml list-audience-generation-jobs` | List audience generation jobs |
| `cleanrooms list-collaborations` | List Clean Rooms collaborations |

## Gotchas: AWS Clean Rooms ML

- Clean Rooms ML operates WITHIN a Clean Rooms collaboration. You must have an active collaboration before using ML features.
- Training datasets must conform to specific schema requirements. Column types and formats are strictly validated during training.
- Lookalike models require a seed audience of sufficient size (minimum varies by region). Too-small seed audiences produce poor results or fail.
- Privacy budgets are consumed per audience generation job. Once exhausted, no more audience generation jobs can run until the budget refreshes or is increased.
- Configured audience models must be associated with a collaboration before audience generation. The association defines output constraints.
- Differential privacy adds noise to outputs. Higher epsilon values reduce noise but weaken privacy guarantees. This is a fundamental tradeoff.
- S3 data must be in the same region as the Clean Rooms collaboration. Cross-region data access is not supported.

## Anti-hallucination rules

1. Always cite specific ARNs, job IDs, or API responses as evidence.
2. Clean Rooms ML and Clean Rooms are separate services. Never conflate their APIs.
3. Privacy budgets are finite resources. Never suggest they are unlimited or auto-replenishing.
4. Differential privacy epsilon values have specific mathematical meaning. Never invent epsilon recommendations.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Training | A1–A2 | Training model failures, configured model errors |
| B — Audience | B1–B2 | Audience generation issues, lookalike model errors |
| C — Collaboration | C1–C2 | Collaboration ML config, privacy budget issues |
| D — Seed & Output | D1–D2 | Seed audience errors, output configuration |
| E — Access | E1–E2 | S3 data access, IAM permissions |
| F — Privacy | F1 | Differential privacy |
| Z — Catch-All | Z1 | General troubleshooting |
