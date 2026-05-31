---
title: "B1 — Solution Training Failures"
description: "Diagnose solution version training failures"
status: active
severity: HIGH
triggers: ["training failed", "solution version error", "CreateSolutionVersion failed"]
owner: devops-agent
objective: "Identify and resolve solution training failures"
context: "Solution versions are trained models. Training takes 1-6+ hours. Requires sufficient data, valid recipe, and proper configuration."
---

## Phase 1 — Triage

MUST:
- Check solution version: `aws personalize describe-solution-version --solution-version-arn <arn>`
- Review failureReason if status is CREATE FAILED
- Verify dataset has sufficient data
- Check recipe compatibility with data

SHOULD:
- Verify hyperparameter configuration
- Check training mode (FULL vs UPDATE)
- Review solution configuration

MAY:
- Check CloudTrail for CreateSolutionVersion events
- Compare with previous successful training

## Phase 2 — Remediate

MUST:
- Ensure minimum data requirements met
- Use compatible recipe for dataset type
- Fix any configuration errors

SHOULD:
- Start with default hyperparameters
- Use FULL training mode for first version

MAY:
- Implement training pipeline with validation

## Common Issues

- symptoms: "Training fails with data error"
  diagnosis: "Insufficient data or data quality issues."
  resolution: "Add more data. Meet minimum requirements."

- symptoms: "Training takes excessively long"
  diagnosis: "Large dataset or complex recipe."
  resolution: "This is normal. Training can take 6+ hours for large datasets."

## Output Format

```yaml
root_cause: "solution_training — <specific_cause>"
evidence:
  - type: solution_version
    content: "<solution version details>"
severity: HIGH
mitigation:
  immediate: "Fix data or configuration"
  long_term: "Implement training monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — training uses user behavioral data
- Service Disruption Risk: LOW — training failures do not affect running campaigns
- Compliance Impact: MEDIUM — model training must comply with data usage policies

## Escalation Conditions
- Solution training fails repeatedly
- Training takes significantly longer than expected
- Trained model produces poor quality recommendations
- Training costs exceed budget thresholds
- Training data insufficient for selected recipe

## Data Sensitivity
- Training data — RESTRICTED
- Model artifacts — CONFIDENTIAL
- Training metrics — INTERNAL
- Hyperparameter configurations — INTERNAL
- Never expose training data or model internals

## Prohibited Actions
- Do NOT train models on data without proper consent
- Do NOT ignore training failure patterns
- Do NOT deploy models without quality validation
- Do NOT train with excessive hyperparameter ranges without cost approval
- Do NOT share model artifacts with unauthorized parties
- Do NOT bypass model evaluation before deployment

## Phase 3 — Rollback
- Revert to previous solution version
- Restore training configurations
- Re-train with corrected parameters
- Verify model quality after re-training
- Restore previous campaign to use older model version
- Document training issues and recipe selection rationale

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
