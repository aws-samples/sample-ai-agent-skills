---
title: "B2 — Model Evaluation Issues"
description: "Diagnose model evaluation metrics, accuracy, and performance issues"
status: active
severity: MEDIUM
triggers: ["poor accuracy", "low precision", "low recall", "model not accurate", "evaluation metrics"]
owner: devops-agent
objective: "Understand and improve model evaluation metrics"
context: "Model evaluation provides precision, recall, and F1 score. Metrics depend on dataset quality, size, and label accuracy. A separate test dataset provides more reliable evaluation."
---

## Phase 1 — Triage

MUST:
- Get model evaluation metrics: `aws lookoutvision describe-model --project-name <project> --model-version <version>`
- Check dataset sizes: `aws lookoutvision describe-dataset --project-name <project> --dataset-type train`
- Verify if a separate test dataset was used: `aws lookoutvision describe-dataset --project-name <project> --dataset-type test`

SHOULD:
- Review precision, recall, and F1 score values
- Check if the training dataset has balanced classes
- Verify image quality consistency across the dataset

MAY:
- Compare metrics across model versions
- Review sample images for labeling accuracy

## Phase 2 — Remediate

MUST:
- Ensure dataset has sufficient images (100+ per class recommended for production)
- Verify all labels are accurate — mislabeled images degrade metrics
- Use a separate test dataset for more reliable evaluation

SHOULD:
- Improve image quality consistency (lighting, angle, resolution)
- Add more diverse examples of both normal and anomaly conditions
- Review and correct any mislabeled images

MAY:
- Train multiple model versions with different dataset compositions
- Adjust confidence threshold based on precision/recall tradeoff

## Common Issues

- symptoms: "Model has high precision but low recall"
  diagnosis: "Model misses anomalies — insufficient anomaly examples in training data."
  resolution: "Add more diverse anomaly images. Ensure anomaly images cover all defect types and severities."

- symptoms: "Model has high recall but low precision"
  diagnosis: "Model flags normal items as anomalies — normal images not diverse enough."
  resolution: "Add more diverse normal images covering all acceptable variations. Check for mislabeled normal images."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision describe-model: GREEN — read-only metrics retrieval"
  - "lookoutvision describe-dataset: GREEN — read-only dataset details"
  - "lookoutvision create-model: YELLOW — initiates new training, incurs charges"
  - "Dataset modification: YELLOW — affects future training results"
```

## Escalation Conditions

- Model metrics are consistently poor despite adequate dataset size and quality
- Business requires accuracy levels that cannot be achieved with current data
- Evaluation metrics do not match real-world performance
- Multiple model versions show similar poor performance

## Data Sensitivity

- `aws lookoutvision describe-model`: MEDIUM — reveals model performance metrics
- Dataset statistics: LOW — aggregate counts and distributions
- Training images: HIGH — may contain proprietary manufacturing images
- Model comparison data: LOW — performance statistics

## Prohibited Actions

- NEVER deploy a model to production without reviewing evaluation metrics
- NEVER assume evaluation metrics represent real-world performance without validation
- NEVER lower confidence thresholds without understanding the false positive impact
- NEVER share model evaluation metrics publicly without authorization

## Phase 3 — Rollback

1. If a new model version has worse metrics, keep the previous version active
2. If dataset was modified and metrics degraded, restore original dataset and retrain
3. If confidence threshold was changed, restore previous value
4. Document metric comparisons across versions for model lineage
5. Verify production inference uses the best-performing model version

## Output Format

```yaml
root_cause: "model_evaluation — <specific_cause>"
evidence:
  - type: evaluation_metrics
    content: "<precision, recall, F1, dataset sizes>"
severity: MEDIUM
mitigation:
  immediate: "Improve dataset quality and retrain"
  long_term: "Establish model performance baselines and monitoring"
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
