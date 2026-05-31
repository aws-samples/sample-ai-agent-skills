---
name: lookout-vision-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Lookout for
  Vision problems by analyzing project creation, dataset imports,
  model training, model evaluation, inference errors, edge deployment,
  image quality, labeling errors, S3 permissions, model packaging,
  Greengrass integration, and following structured runbooks.
  Activate when: project creation failures, dataset import errors,
  model training failures, model evaluation issues, inference errors,
  edge deployment failures, image quality issues, labeling errors,
  S3 permission problems, model packaging errors, Greengrass
  integration issues, or the user says something is wrong with
  Lookout for Vision.
compatibility: >
  Requires AWS CLI or SDK access with lookoutvision, s3, iam,
  greengrass, kms, cloudtrail, and cloudwatch permissions.
---

# Amazon Lookout for Vision Diagnostics

## When to use

Any Amazon Lookout for Vision investigation — project creation, dataset imports, model training, model evaluation, inference errors, edge deployment, image quality, labeling, S3 permissions, model packaging, or Greengrass integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws lookoutvision list-projects --query 'Projects[*].{Name:ProjectName,Arn:ProjectArn,CreatedAt:CreationTimestamp}'
aws lookoutvision list-models --project-name <project> --query 'Models[*].{Version:ModelVersion,Arn:ModelArn,Status:Status}'
aws lookoutvision list-datasets --project-name <project>
```

### Step 2 — Domain deep dive

```
aws lookoutvision describe-model --project-name <project> --model-version <version>
aws lookoutvision describe-dataset --project-name <project> --dataset-type <train|test>
aws s3 ls s3://<dataset-bucket>/<prefix>/
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutvision.amazonaws.com --max-results 20
aws lookoutvision list-model-packaging-jobs --project-name <project>
aws greengrass list-components --query 'components[*].{Name:componentName,Arn:arn}'
```

Read `references/guardrails.md` before concluding on any Lookout for Vision issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `lookoutvision list-projects` | List projects |
| `lookoutvision describe-model` | Get model details and metrics |
| `lookoutvision describe-dataset` | Get dataset details |
| `lookoutvision list-model-packaging-jobs` | List edge packaging jobs |
| `lookoutvision detect-anomalies` | Run inference on an image |
| `lookoutvision start-model` | Start model for inference |
| `lookoutvision stop-model` | Stop model to save costs |

## Gotchas: Amazon Lookout for Vision

- Models must be explicitly started before inference. A trained model in TRAINED status cannot serve predictions until started (HOSTED status).
- Running models incur hourly charges. Always stop models when not in use to avoid unexpected costs.
- Training requires a minimum of 20 normal images and 10 anomaly images. Fewer images produce unreliable models.
- Image quality significantly affects model accuracy. Consistent lighting, angle, and resolution are critical.
- Edge deployment requires AWS IoT Greengrass V2. The model must be packaged as a Greengrass component before deployment.
- Dataset labels must be accurate. Mislabeled images (normal labeled as anomaly or vice versa) degrade model performance significantly.
- Lookout for Vision is regional. Projects, models, and datasets are bound to the region where they were created.

## Anti-hallucination rules

1. Always cite specific project names, model versions, or API responses as evidence.
2. Model training and inference are separate operations. A trained model must be started before it can serve predictions.
3. Edge deployment requires Greengrass V2. Never suggest direct edge deployment without Greengrass.
4. Image quality requirements are strict. Never claim any image quality will work.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Project & Data | A1–A2 | Project creation failures, dataset import errors |
| B — Training | B1–B2 | Model training failures, model evaluation issues |
| C — Inference | C1–C2 | Inference errors, edge deployment failures |
| D — Quality | D1–D2 | Image quality issues, labeling errors |
| E — Access | E1–E2 | S3 permissions, model packaging |
| F — Integration | F1 | Greengrass integration |
| Z — Catch-All | Z1 | General troubleshooting |
