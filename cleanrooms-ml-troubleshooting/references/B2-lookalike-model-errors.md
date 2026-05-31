---
title: "B2 — Lookalike Model Errors"
description: "Diagnose lookalike model creation, training, and inference errors"
status: active
severity: HIGH
triggers: ["lookalike model failed", "lookalike error", "similar audience error", "model quality poor"]
owner: devops-agent
objective: "Resolve lookalike model errors and improve audience quality"
context: "Lookalike models find audiences similar to a seed audience. They require sufficient seed data, proper training, and appropriate similarity thresholds."
---

## Phase 1 — Triage

MUST:
- Get audience model details: `aws cleanroomsml get-audience-model --audience-model-arn <arn>`
- Check audience model status for ACTIVE or CREATE_FAILED
- Verify training dataset used by the model: `aws cleanroomsml get-training-dataset --training-dataset-arn <arn>`

SHOULD:
- Check seed audience size meets minimum requirements
- Review model metrics if available in the audience model details

MAY:
- Compare seed audience schema with training dataset schema
- Review CloudTrail for model creation events

## Phase 2 — Remediate

MUST:
- Ensure seed audience has minimum 500+ records with matching schema
- Verify training dataset is ACTIVE and was trained on representative data
- Fix schema alignment between seed audience and training data

SHOULD:
- Increase seed audience size for better lookalike quality
- Validate seed audience data quality — remove duplicates and invalid records

MAY:
- Retrain the audience model with a larger or more diverse training dataset
- Adjust audience generation parameters for different similarity thresholds

## Common Issues

- symptoms: "Audience model status is CREATE_FAILED"
  diagnosis: "Training dataset has insufficient data or schema errors."
  resolution: "Verify training dataset has adequate records and correct schema. Recreate audience model after fixing training data."

- symptoms: "Lookalike audience quality is poor — low overlap with expected segments"
  diagnosis: "Seed audience too small or not representative of target segment."
  resolution: "Increase seed audience to 1000+ records. Ensure seed audience accurately represents the target segment characteristics."

## Safety Ratings

```
safety_ratings:
  - "get-audience-model: GREEN — read-only detail retrieval"
  - "get-training-dataset: GREEN — read-only detail retrieval"
  - "create-audience-model: YELLOW — initiates model training, consumes compute"
  - "delete-audience-model: RED — permanently removes model and training artifacts"
```

## Escalation Conditions

- Audience model creation fails repeatedly with internal errors after data fixes
- Model quality consistently poor despite adequate seed audience size
- Training takes over 8 hours without status change
- Multiple collaboration members report different model behavior

## Data Sensitivity

- `aws cleanroomsml get-audience-model`: MEDIUM — reveals model configuration and training parameters
- `aws cleanroomsml get-training-dataset`: MEDIUM — reveals schema and data source locations
- Seed audience data: HIGH — contains member-contributed audience segments
- Model training artifacts: HIGH — derived from collaboration member data

## Prohibited Actions

- NEVER inspect raw seed audience data content — it belongs to the contributing collaboration member
- NEVER delete an audience model that has active configured audience models referencing it
- NEVER share model quality metrics outside the collaboration without member consent
- NEVER retrain models without verifying training data governance requirements

## Phase 3 — Rollback

1. If an audience model was created with wrong parameters, delete it: `aws cleanroomsml delete-audience-model --audience-model-arn <arn>` (only if no configured models reference it)
2. If training dataset was corrupted, delete and recreate: `aws cleanroomsml delete-training-dataset --training-dataset-arn <arn>`
3. Verify no audience generation jobs are in progress before deleting models
4. Notify collaboration members of model changes
5. Document the issue and resolution for model lineage tracking

## Output Format

```yaml
root_cause: "lookalike_model_error — <specific_cause>"
evidence:
  - type: model_error
    content: "<error details from get-audience-model>"
severity: HIGH
mitigation:
  immediate: "Fix data quality and retrain model"
  long_term: "Implement seed audience validation pipeline"
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
