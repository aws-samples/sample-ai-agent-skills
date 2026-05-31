---
title: "B1 — Model Training Failures"
description: "Diagnose Amazon Lookout for Equipment model training failures"
status: active
severity: HIGH
triggers: ["model training failed", "CreateModel error", "training stuck", "model status FAILED"]
owner: devops-agent
objective: "Resolve model training failures and produce a working anomaly detection model"
context: "Model training uses ingested sensor data to build anomaly detection models. Training requires minimum 14 days of data and can take hours to days."
---

## Phase 1 — Triage

MUST:
- Check model status: `aws lookoutequipment describe-model --model-name <name>`
- Verify dataset has ingested data: `aws lookoutequipment describe-dataset --dataset-name <name>`
- Check training data time range covers minimum 14 days
- Verify IAM role permissions for the model training job

SHOULD:
- Check CloudTrail for CreateModel errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateModel --max-results 10`
- Verify the training/evaluation split time ranges are valid and non-overlapping
- Check service quotas for concurrent training jobs

MAY:
- Review the dataset schema for sensor columns that may cause issues
- Check if labels were provided and cover the training time range

## Phase 2 — Remediate

MUST:
- Ensure training data covers at least 14 days (90+ days recommended)
- Fix training and evaluation time range parameters to be valid ISO 8601 ranges
- Ensure evaluation period does not overlap with training period
- Verify the IAM role has access to the S3 data used during ingestion

SHOULD:
- Increase training data to 90+ days for production-quality models
- Add labels for known anomaly periods to improve model accuracy
- Ensure sensor data has minimal missing values during the training window

MAY:
- Experiment with different training/evaluation time splits
- Remove noisy or irrelevant sensor columns from the schema

## Common Issues

- symptoms: "Model status is FAILED with insufficient data error"
  diagnosis: "Training data does not meet the 14-day minimum requirement."
  resolution: "Ingest additional sensor data covering at least 14 continuous days. Re-create the model with the expanded time range."

- symptoms: "Model training has been in TRAINING status for over 48 hours"
  diagnosis: "Large datasets with many sensors can take extended training time. This may be normal."
  resolution: "Wait up to 72 hours for large datasets. If still TRAINING after 72 hours, contact AWS Support."

- symptoms: "Model FAILED with 'unable to access data' error"
  diagnosis: "The IAM role lost access to the S3 training data after ingestion."
  resolution: "Verify the IAM role still has s3:GetObject on the original data path. Check if bucket policy or S3 Block Public Access changed."

## Safety Ratings

```
safety_ratings:
  - "describe-model: GREEN — read-only model metadata retrieval"
  - "describe-dataset: GREEN — read-only dataset metadata"
  - "CreateModel: YELLOW — initiates long-running training job, consumes compute"
  - "delete-model: RED — permanently removes model and associated inference schedulers"
```

## Escalation Conditions

- Model training fails repeatedly with internal service errors after data and permission fixes
- Training job exceeds 72 hours without completion or failure
- Model trains successfully but produces unusable results (all anomalies or no anomalies)
- Urgent production need for a working model with limited historical data

## Data Sensitivity

- `aws lookoutequipment describe-model`: LOW — returns model configuration and metrics
- Model training metrics: LOW — contains statistical measures, not raw sensor data
- Training time ranges: LOW — contains date ranges only
- Label data: MEDIUM — may reveal proprietary knowledge about equipment failure patterns

## Prohibited Actions

- NEVER cancel a training job that has been running for less than 24 hours — training times vary significantly
- NEVER suggest training with less than 14 days of data as a workaround
- NEVER delete a model that has active inference schedulers without stopping them first
- NEVER share model evaluation metrics externally without confirming data classification

## Phase 3 — Rollback

1. If a model was created with wrong parameters, delete it: `aws lookoutequipment delete-model --model-name <name>` (only after stopping any inference schedulers)
2. If training consumed excessive resources, check billing and set up budget alerts
3. If labels were incorrectly applied, recreate the model without labels or with corrected labels
4. Document the successful training configuration for future model iterations
5. Notify the ML team of any model deletions to update deployment pipelines

## Output Format

```yaml
root_cause: "model_training — <specific_cause>"
evidence:
  - type: training_error
    content: "<error details from describe-model>"
severity: HIGH
mitigation:
  immediate: "Fix data or configuration issues and retrain"
  long_term: "Establish minimum data quality standards before training"
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
