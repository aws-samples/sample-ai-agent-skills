---
title: "B1 — Model Training Failures"
description: "Diagnose model training failures and training configuration issues"
status: active
severity: HIGH
triggers: ["training failed", "model training error", "TRAINING_FAILED", "cannot train model"]
owner: devops-agent
objective: "Resolve model training failures and produce a trained model"
context: "Model training is asynchronous and can take 30 minutes to several hours. It requires a dataset with minimum 20 normal and 10 anomaly images. Training produces a model version."
---

## Phase 1 — Triage

MUST:
- Check model status: `aws lookoutvision describe-model --project-name <project> --model-version <version>`
- Verify dataset is complete: `aws lookoutvision describe-dataset --project-name <project> --dataset-type train`
- Check CloudTrail for training errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutvision.amazonaws.com --max-results 20`

SHOULD:
- Verify dataset has minimum images (20 normal, 10 anomaly)
- Check IAM role permissions for S3 access to training data
- Verify S3 bucket with training data is accessible

MAY:
- Review training output location in S3 for error logs
- Check service quotas for concurrent training jobs

## Phase 2 — Remediate

MUST:
- Ensure dataset meets minimum size requirements (20 normal, 10 anomaly)
- Fix IAM role permissions for S3 and KMS access
- Verify all images in the dataset are valid (correct format, minimum size)

SHOULD:
- Increase dataset size to 100+ images per class for better results
- Ensure image quality is consistent across the dataset
- Use a separate test dataset for more accurate evaluation

MAY:
- Train with a subset of data first to validate configuration
- Review and fix any mislabeled images before retraining

## Common Issues

- symptoms: "Model status is TRAINING_FAILED"
  diagnosis: "Insufficient training data or IAM permission errors."
  resolution: "Check dataset size (minimum 20 normal + 10 anomaly). Verify IAM role has s3:GetObject on training data bucket."

- symptoms: "Training starts but fails after several minutes"
  diagnosis: "Image quality issues — corrupted files, unsupported formats, or too-small images."
  resolution: "Validate all images are PNG/JPEG, at least 64x64 pixels, and not corrupted. Remove problematic images and retrain."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision describe-model: GREEN — read-only model details"
  - "lookoutvision describe-dataset: GREEN — read-only dataset details"
  - "lookoutvision create-model: YELLOW — initiates training, incurs compute charges"
  - "lookoutvision delete-model: RED — permanently removes model version"
```

## Escalation Conditions

- Training fails repeatedly with internal errors after fixing data and permissions
- Training takes over 6 hours without status change
- Service quota for concurrent training jobs is reached
- Training succeeds but model metrics are unacceptably poor

## Data Sensitivity

- `aws lookoutvision describe-model`: MEDIUM — reveals model configuration and metrics
- `aws lookoutvision describe-dataset`: MEDIUM — reveals dataset statistics
- Training images in S3: HIGH — may contain proprietary manufacturing images
- Model evaluation metrics: LOW — performance statistics

## Prohibited Actions

- NEVER delete a model version that is currently serving inference (HOSTED status)
- NEVER modify training data while a training job is in progress
- NEVER train models with mislabeled data — fix labels first
- NEVER share model metrics outside the organization without authorization

## Phase 3 — Rollback

1. If training produced a poor model, keep it for comparison but do not start it
2. If training data was modified incorrectly, restore from backup and retrain
3. If IAM role was changed, restore previous permissions
4. Previous model versions remain available — switch back if new version is worse
5. Document the training failure and root cause

## Output Format

```yaml
root_cause: "model_training — <specific_cause>"
evidence:
  - type: training_error
    content: "<error details from describe-model or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix data/permissions and retrain"
  long_term: "Implement dataset validation and training pipeline"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
