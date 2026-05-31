---
title: "C1 — Custom Classifier Training Issues"
description: "Diagnose custom document classifier training failures"
status: active
severity: HIGH
triggers:
  - "classifier training failed"
  - "CreateDocumentClassifier error"
  - "training data error"
owner: devops-agent
objective: "Identify and resolve classifier training failures"
context: "Custom classifiers categorize documents into custom classes. Training requires labeled data in CSV or augmented manifest format. Multi-class and multi-label modes available."
---

## Phase 1 — Triage

MUST:
- Check classifier status: `aws comprehend describe-document-classifier --document-classifier-arn <arn>`
- Verify training data in S3 is accessible
- Check training data format (CSV: label,text per line)
- Review status message for failure reason

SHOULD:
- Verify minimum documents per class (10+ for multi-class)
- Check class distribution balance
- Verify IAM data access role permissions

MAY:
- Review training metrics after completion
- Check CloudTrail for CreateDocumentClassifier events

## Phase 2 — Remediate

MUST:
- Fix training data format (CSV or augmented manifest)
- Provide minimum documents per class
- Grant S3 access to data access role

SHOULD:
- Balance class distribution
- Include test dataset for evaluation
- Use diverse examples per class

MAY:
- Implement training data validation pipeline

## Common Issues

- symptoms: "Training fails with data format error"
  diagnosis: "CSV format incorrect or encoding issues."
  resolution: "Use UTF-8 CSV with label,text format. No headers."

- symptoms: "Training completes but accuracy is low"
  diagnosis: "Insufficient or unbalanced training data."
  resolution: "Add more diverse examples. Balance class distribution."

## Output Format

```yaml
root_cause: "classifier_training — <specific_cause>"
evidence:
  - type: classifier_status
    content: "<classifier details and metrics>"
  - type: training_data
    content: "<training data statistics>"
severity: HIGH
mitigation:
  immediate: "Fix training data"
  long_term: "Implement training data quality pipeline"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Training data contains labeled documents that may include sensitive content |
| Blast Radius | MEDIUM — Failed training wastes compute resources; biased models affect downstream decisions |
| Rollback Complexity | MEDIUM — Retraining requires time; previous model versions should be preserved |
| Compliance Scope | HIGH — Classification models used for content moderation or compliance must meet accuracy standards |

## Escalation Conditions

- Escalate to data governance if training data contains PII/PHI without proper handling controls
- Escalate to management if repeated training failures consume significant compute budget
- Escalate to compliance if classifier accuracy falls below required thresholds for regulated use cases
- Escalate to AWS Support if training fails with internal service errors

## Data Sensitivity

- **Training data** in S3 contains labeled documents — enforce encryption and access controls
- **Model artifacts** encode document patterns — treat as sensitive if trained on sensitive data
- **IAM data access role** has S3 read/write access — use least-privilege scoping
- **Training metrics** (precision, recall, F1) should be documented for audit purposes

## PROHIBITED Actions

- Do NOT use unbalanced or biased training data for classifiers making decisions about people
- Do NOT deploy classifiers to production without documenting accuracy metrics
- Do NOT store training data in unencrypted S3 buckets
- Do NOT grant the data access role permissions beyond the specific training bucket
- Do NOT delete training data or model artifacts that may be needed for compliance audits

## Phase 3 — Rollback

- If new classifier performs poorly, revert downstream systems to use the previous classifier ARN
- Delete the poorly performing classifier: `aws comprehend delete-document-classifier --document-classifier-arn <arn>`
- Restore training data from S3 versioned backup if data was corrupted
- Document the rollback and root cause for model governance records

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
