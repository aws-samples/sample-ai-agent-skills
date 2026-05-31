---
title: "C2 — ML Matching Accuracy"
description: "Diagnose ML matching accuracy issues including data quality, confidence thresholds, and model performance"
status: active
severity: MEDIUM
triggers: ["ML matching poor accuracy", "low confidence scores", "ML matching not working well", "machine learning matching quality"]
owner: devops-agent
objective: "Improve ML matching accuracy through data quality and configuration optimization"
context: "ML matching uses machine learning for probabilistic matching. Accuracy depends on data quality, volume, and field completeness. Confidence scores indicate match probability. Small or poor-quality datasets produce unreliable results."
---

## Phase 1 — Triage

MUST:
- Get workflow configuration: `aws entityresolution get-matching-workflow --workflow-name <workflow-name>`
- Review recent job results and confidence score distribution
- Assess input data quality (completeness, consistency, volume)

SHOULD:
- Sample high-confidence and low-confidence matches for manual review
- Check if input data meets minimum volume requirements
- Review schema mapping to ensure relevant fields are mapped

MAY:
- Analyze field completeness rates across the dataset
- Compare ML matching results with rule-based matching on the same data
- Review data preprocessing and normalization

## Phase 2 — Remediate

MUST:
- Improve input data quality (fill missing fields, standardize formats)
- Ensure sufficient data volume for ML model effectiveness
- Map all relevant fields in the schema mapping

SHOULD:
- Adjust confidence threshold based on precision/recall requirements
- Standardize data formatting before matching (addresses, names, etc.)
- Validate results with known match pairs

MAY:
- Implement data preprocessing pipeline before Entity Resolution
- Consider rule-based matching for deterministic requirements
- Create feedback loops to improve data quality over time

## Safety Ratings

```
safety_ratings:
  - "get-matching-workflow: GREEN — read-only workflow configuration"
  - "get-matching-job: GREEN — read-only job results"
  - "update-matching-workflow: YELLOW — modifies ML configuration"
  - "start-matching-job: YELLOW — runs ML matching job"
```

## Escalation Conditions

- ML matching accuracy below business requirements despite data improvements
- Confidence scores are uniformly low across all matches
- ML matching produces inconsistent results across runs
- Business requires deterministic matching that ML cannot guarantee

## Data Sensitivity

- `aws entityresolution get-matching-workflow`: LOW — workflow configuration
- ML matching results: HIGH — contains matched PII records with confidence scores
- Input data: HIGH — contains PII for matching
- Confidence scores: LOW — numerical values

## Prohibited Actions

- NEVER deploy ML matching results to production without validation
- NEVER set confidence threshold to zero to get more matches
- NEVER assume ML matching is 100% accurate
- NEVER use ML matching for compliance-critical exact matching requirements

## Phase 3 — Rollback

1. If ML configuration changes degraded accuracy, restore previous settings
2. Re-run the matching job with restored configuration
3. Compare results to verify improvement
4. If ML matching is unsuitable, switch to rule-based matching
5. Document accuracy metrics and configuration changes

## Common Issues

- symptoms: "ML matching produces very few matches with low confidence"
  diagnosis: "Input data has too many missing fields or insufficient volume for ML model."
  resolution: "Improve data completeness. Ensure minimum data volume. Map more relevant fields in schema mapping."

- symptoms: "ML matching produces many false positives"
  diagnosis: "Confidence threshold is too low, or input data has many similar but distinct records."
  resolution: "Increase confidence threshold. Add more distinguishing fields to schema mapping."

- symptoms: "ML matching results vary significantly between runs"
  diagnosis: "Input data changes between runs, or data volume is at the edge of ML model requirements."
  resolution: "Ensure consistent input data. Increase data volume. Consider rule-based matching for consistency."

## Output Format

```yaml
root_cause: "ml_matching_accuracy — <specific_cause>"
evidence:
  - type: accuracy_analysis
    content: "<confidence score distribution and match quality metrics>"
severity: MEDIUM
mitigation:
  immediate: "Adjust ML configuration or data quality"
  long_term: "Implement data quality pipeline and matching validation"
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
