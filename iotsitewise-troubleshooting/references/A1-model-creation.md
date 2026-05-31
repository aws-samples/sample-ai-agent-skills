---
title: "A1 — Asset Model Creation Failures"
description: "Diagnose asset model creation and update failures"
status: active
severity: HIGH
triggers:
  - "model creation failed"
  - "model validation error"
  - "cannot create model"
  - "model stuck in CREATING"
owner: devops-agent
objective: "Identify and resolve asset model creation failures"
context: "Model creation fails when: property names duplicate, formula references invalid, hierarchy circular, or account limits reached."
---

## Phase 1 — Triage

MUST:
- Check model status: `aws iotsitewise describe-asset-model --asset-model-id <id>`
- List existing models: `aws iotsitewise list-asset-models`
- Verify property definitions for duplicates and valid types
- Check formula references in transforms and metrics

SHOULD:
- Verify hierarchy definitions reference valid child models
- Check account limits for number of models
- Review CloudTrail for creation errors

MAY:
- Validate model JSON/YAML before creation
- Check for naming conflicts with existing models

## Phase 2 — Remediate

MUST:
- Fix duplicate property names
- Correct invalid formula references
- Remove circular hierarchy definitions

SHOULD:
- Use unique, descriptive property names
- Test formulas with sample data before model creation

## Output Format

```yaml
root_cause: "model_creation — <specific_cause>"
evidence:
  - type: model_status
    content: "<model details and error>"
severity: HIGH
mitigation:
  immediate: "Fix model definition errors"
  long_term: "Implement model validation before deployment"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
