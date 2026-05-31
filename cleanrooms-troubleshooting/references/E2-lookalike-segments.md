---
title: "E2 — Lookalike Segment Issues"
description: "Diagnose Clean Rooms ML lookalike segment generation problems"
status: active
severity: MEDIUM
triggers:
  - "lookalike segment"
  - "lookalike model"
  - "audience generation"
  - "similar audience"
  - "segment generation"
owner: devops-agent
objective: "Resolve lookalike segment generation issues"
context: "Clean Rooms ML lookalike modeling generates audience segments similar to a seed audience. It requires a trained model and seed data. Common issues include model not trained, seed data format errors, segment size configuration, and output delivery failures."
---

## Phase 1 — Triage

MUST:
- Check configured audience model: `aws cleanrooms-ml list-configured-audience-models`
- Verify the audience model is trained and active
- Check seed data configuration and format
- Verify output configuration for segment delivery

SHOULD:
- Check if the seed audience meets minimum size requirements
- Verify the model was trained on compatible data
- Check IAM permissions for audience generation

MAY:
- List audience generation jobs: `aws cleanrooms-ml list-audience-generation-jobs`
- Check job status and error details

## Phase 2 — Remediate

MUST:
- If model not trained, complete model training first
- Fix seed data format to match model requirements
- Ensure output S3 bucket is accessible

SHOULD:
- Verify seed audience size meets minimum thresholds
- Test with a small segment before generating large audiences

## Output Format

```yaml
root_cause: "lookalike_segments — <specific_cause>"
evidence:
  - type: audience_model
    content: "<configured audience model details>"
  - type: generation_job
    content: "<audience generation job status>"
severity: MEDIUM
mitigation:
  immediate: "Fix model, seed data, or output configuration"
  long_term: "Automate lookalike segment generation pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Complete model training | GREEN |
| Fix seed data format | GREEN |
| Ensure output S3 bucket is accessible | YELLOW |
| Verify seed audience size | GREEN |
| Test with small segment | GREEN |

## Escalation Conditions

- Fix requires modifying production audience model configurations
- Fix involves changing segment output destinations

## Data Sensitivity

- **HIGH**: lookalike segments contain audience data derived from collaborative modeling
- Seed data and generated segments reveal targeting strategies and audience characteristics

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If audience model was misconfigured, delete and recreate with correct settings
2. If output configuration was changed, restore previous S3 destination
3. If seed data was modified, restore previous seed dataset
4. Verify rollback by confirming audience generation job configuration matches expected state

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
