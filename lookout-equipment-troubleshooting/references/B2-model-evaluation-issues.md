---
title: "B2 — Model Evaluation Issues"
description: "Diagnose Amazon Lookout for Equipment model evaluation and quality issues"
status: active
severity: MEDIUM
triggers: ["poor model performance", "too many false positives", "model not detecting anomalies", "evaluation metrics bad"]
owner: devops-agent
objective: "Diagnose model quality issues and improve anomaly detection accuracy"
context: "Model evaluation metrics indicate how well the model detects anomalies. Poor metrics may indicate insufficient data, poor data quality, or incorrect labeling."
---

## Phase 1 — Triage

MUST:
- Get model details and metrics: `aws lookoutequipment describe-model --model-name <name>`
- Check the training and evaluation time ranges used
- Verify the dataset had sufficient data (14+ days minimum, 90+ recommended)
- Check if labels were provided during training

SHOULD:
- Compare evaluation metrics across multiple model versions if available
- Check data quality during the evaluation period for gaps or anomalies
- Verify sensor data was representative of normal operating conditions during training

MAY:
- List all models for the dataset to compare: `aws lookoutequipment list-models --dataset-name-begins-with <prefix>`
- Review the sensor contribution scores from recent inference results

## Phase 2 — Remediate

MUST:
- If insufficient data: ingest more data (target 90+ days) and retrain
- If no labels provided: add labels for known anomaly periods and retrain
- If evaluation period had anomalies: adjust evaluation time range to normal operation period
- Verify training data represents normal equipment behavior (not during maintenance or shutdown)

SHOULD:
- Retrain with labels marking known good and bad periods
- Ensure training data covers all normal operating modes (startup, steady-state, shutdown)
- Remove sensors with constant values or excessive noise from the schema

MAY:
- Train multiple models with different time ranges and compare
- Consult domain experts to identify which sensors are most relevant

## Common Issues

- symptoms: "Model detects anomalies constantly (high false positive rate)"
  diagnosis: "Training data likely contained anomalous periods that the model learned as normal, or insufficient normal data."
  resolution: "Retrain with clean normal-operation data. Add labels to mark anomalous periods in training data so the model can distinguish them."

- symptoms: "Model never detects anomalies (high false negative rate)"
  diagnosis: "The anomaly patterns in production differ from what the model learned, or the model was trained on data that already included the anomaly pattern."
  resolution: "Retrain with data that clearly separates normal from anomalous periods. Use labels to improve detection sensitivity."

- symptoms: "Sensor contribution scores are evenly distributed across all sensors"
  diagnosis: "The model cannot identify specific sensors driving anomalies. Data may lack clear anomaly signatures."
  resolution: "Review sensor data quality. Ensure sensors are properly calibrated and producing meaningful readings."

## Safety Ratings

```
safety_ratings:
  - "describe-model: GREEN — read-only model metadata and metrics"
  - "list-models: GREEN — read-only enumeration"
  - "Retraining with new parameters: YELLOW — creates new model version, consumes compute"
  - "Modifying labels: YELLOW — affects future model training quality"
```

## Escalation Conditions

- Model consistently produces poor metrics despite sufficient data and correct labels
- Business-critical equipment monitoring depends on model accuracy improvements
- Domain experts disagree on what constitutes normal vs anomalous sensor readings
- Model quality degrades over time suggesting concept drift in equipment behavior

## Data Sensitivity

- `aws lookoutequipment describe-model`: LOW — returns model metrics and configuration
- Evaluation metrics: LOW — statistical measures without raw data
- Sensor contribution scores: MEDIUM — may reveal equipment behavior patterns
- Label data: MEDIUM — reveals knowledge about equipment failure modes

## Prohibited Actions

- NEVER claim a model is production-ready based solely on training metrics without domain expert review
- NEVER suggest deploying a model with known high false positive rates to production monitoring
- NEVER delete historical models without preserving their evaluation metrics for comparison
- NEVER adjust labels without consulting the equipment maintenance team

## Phase 3 — Rollback

1. If a retrained model performs worse, keep the previous model active and delete the new one
2. If labels were incorrectly modified, restore from backup and retrain
3. If evaluation revealed the model is unsuitable, stop any inference schedulers using it
4. Document all model versions and their evaluation metrics for audit trail
5. Notify operations team if model quality affects production monitoring reliability

## Output Format

```yaml
root_cause: "model_evaluation — <specific_cause>"
evidence:
  - type: evaluation_metrics
    content: "<metrics from describe-model>"
severity: MEDIUM
mitigation:
  immediate: "Retrain with improved data or labels"
  long_term: "Establish model quality baselines and automated retraining"
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
