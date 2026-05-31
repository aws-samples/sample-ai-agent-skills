---
title: "D2 — Labeling Errors"
description: "Diagnose and fix image labeling errors affecting model accuracy"
status: active
severity: HIGH
triggers: ["mislabeled", "wrong label", "labeling error", "label quality", "annotation error"]
owner: devops-agent
objective: "Fix labeling errors and improve dataset quality for model training"
context: "Labels (normal vs. anomaly) must be accurate. Even a small percentage of mislabeled images significantly degrades model performance. Label review is critical before training."
---

## Phase 1 — Triage

MUST:
- Review dataset statistics: `aws lookoutvision describe-dataset --project-name <project> --dataset-type train`
- Check class distribution — severely imbalanced classes may indicate labeling issues
- Review sample images from each class for obvious mislabels

SHOULD:
- Check if labeling was done by multiple people (inconsistency risk)
- Review borderline cases — images that are ambiguous between normal and anomaly
- Verify labeling guidelines were documented and followed

MAY:
- Use the Lookout for Vision console to visually review labeled images
- Check model evaluation for patterns in misclassified images

## Phase 2 — Remediate

MUST:
- Review and correct mislabeled images before retraining
- Establish clear labeling guidelines with examples of normal and anomaly
- Ensure labeling consistency — same criteria applied to all images

SHOULD:
- Have domain experts review labels for accuracy
- Use consensus labeling (multiple reviewers) for ambiguous cases
- Document labeling criteria with visual examples

MAY:
- Implement a labeling review workflow before dataset import
- Use the model's misclassifications to identify potential mislabels

## Common Issues

- symptoms: "Model has poor precision — flags normal items as anomalies"
  diagnosis: "Normal images mislabeled as anomaly in training data."
  resolution: "Review anomaly-labeled images. Correct any that are actually normal. Retrain model."

- symptoms: "Model has poor recall — misses actual anomalies"
  diagnosis: "Anomaly images mislabeled as normal in training data."
  resolution: "Review normal-labeled images. Correct any that are actually anomalous. Retrain model."

## Safety Ratings

```
safety_ratings:
  - "lookoutvision describe-dataset: GREEN — read-only dataset details"
  - "Label correction: YELLOW — changes training data, requires retraining"
  - "Dataset re-import: YELLOW — replaces dataset content"
  - "Model retraining: YELLOW — incurs compute charges"
```

## Escalation Conditions

- Labeling criteria are ambiguous and domain experts disagree
- Large percentage of images need relabeling (>10%)
- Labeling errors were introduced by automated labeling system
- Production model was trained on mislabeled data and is serving incorrect predictions

## Data Sensitivity

- `aws lookoutvision describe-dataset`: MEDIUM — reveals dataset statistics
- Training images: HIGH — may contain proprietary product images
- Labeling criteria: LOW — classification guidelines
- Model misclassification patterns: MEDIUM — reveals model weaknesses

## Prohibited Actions

- NEVER train models with known labeling errors — fix labels first
- NEVER change labels without domain expert review
- NEVER assume automated labels are correct without validation
- NEVER delete labeled images without backup

## Phase 3 — Rollback

1. If labels were changed incorrectly, restore from backup manifest
2. If dataset was re-imported with wrong labels, delete and re-import with correct labels
3. If model was retrained with incorrect labels, revert to previous model version
4. Document labeling corrections for audit trail
5. Verify corrected labels before retraining

## Output Format

```yaml
root_cause: "labeling_error — <specific_cause>"
evidence:
  - type: label_issue
    content: "<mislabel count, class distribution, affected images>"
severity: HIGH
mitigation:
  immediate: "Correct mislabeled images and retrain"
  long_term: "Implement labeling review workflow and guidelines"
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
