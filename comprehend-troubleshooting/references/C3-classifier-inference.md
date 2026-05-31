---
title: "C3 — Custom Classifier Inference Issues"
description: "Diagnose custom classifier inference problems"
status: active
severity: MEDIUM
triggers:
  - "ClassifyDocument error"
  - "wrong classification"
  - "classification confidence low"
owner: devops-agent
objective: "Identify and resolve classifier inference issues"
context: "ClassifyDocument uses an endpoint to classify text. Returns classes with confidence scores. Endpoint must be IN_SERVICE. Text must be within size limits."
---

## Phase 1 — Triage

MUST:
- Verify endpoint is IN_SERVICE: `aws comprehend describe-endpoint --endpoint-arn <arn>`
- Test classification: `aws comprehend classify-document --endpoint-arn <arn> --text "sample text"`
- Review returned classes and confidence scores
- Check text size limits

SHOULD:
- Test with known positive examples from training data
- Compare confidence scores across classes
- Check for class imbalance in results

MAY:
- Monitor endpoint throughput and latency
- Compare with batch classification results

## Phase 2 — Remediate

MUST:
- Ensure endpoint is IN_SERVICE
- Use text within size limits
- Handle multi-class vs multi-label response format

SHOULD:
- Set confidence threshold for classification decisions
- Retrain with more data if accuracy is poor

MAY:
- Implement classification validation pipeline

## Common Issues

- symptoms: "ClassifyDocument returns ResourceUnavailableException"
  diagnosis: "Endpoint not IN_SERVICE or deleted."
  resolution: "Check endpoint status. Create new endpoint if needed."

- symptoms: "Classification always returns same class"
  diagnosis: "Training data heavily imbalanced toward one class."
  resolution: "Retrain with balanced data. Check training metrics."

## Output Format

```yaml
root_cause: "classifier_inference — <specific_cause>"
evidence:
  - type: classification_result
    content: "<ClassifyDocument response>"
  - type: endpoint_status
    content: "<endpoint status>"
severity: MEDIUM
mitigation:
  immediate: "Fix endpoint or retrain model"
  long_term: "Implement classification monitoring and retraining pipeline"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Documents sent for classification may contain sensitive content |
| Blast Radius | LOW — Read-only classification; no data modification |
| Rollback Complexity | LOW — Stateless inference; switch endpoint ARN to revert |
| Compliance Scope | MEDIUM — Classification results may drive automated workflows |

## Escalation Conditions

- Escalate to model owners if classification accuracy degrades below acceptable thresholds
- Escalate to operations if endpoint returns ResourceUnavailableException persistently
- Escalate to compliance if misclassification affects regulated content routing

## Data Sensitivity

- **Input documents** may contain sensitive content — do not log in troubleshooting artifacts
- **Classification results** may be used for content routing — ensure access controls on result storage
- **Confidence scores** should be evaluated before acting on classification decisions

## PROHIBITED Actions

- Do NOT rely on classification results without confidence threshold validation
- Do NOT send production documents to test endpoints or vice versa
- Do NOT ignore low confidence scores for compliance-critical classifications
- Do NOT log input document content in application logs

## Phase 3 — Rollback

- Switch application to use previous endpoint ARN if new model produces poor results
- If endpoint is unavailable, create a new endpoint from the last known good classifier
- Reprocess misclassified documents through the corrected endpoint
- Notify downstream consumers of any misclassification incidents

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
