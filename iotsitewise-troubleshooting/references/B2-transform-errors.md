---
title: "B2 — Transform Computation Errors"
description: "Diagnose transform formula computation failures"
status: active
severity: MEDIUM
triggers:
  - "transform not computing"
  - "formula error"
  - "transform value wrong"
  - "undefined result"
owner: devops-agent
objective: "Identify and resolve transform computation errors"
context: "Transform errors occur when: input property has no value, formula has division by zero, referenced property doesn't exist, or circular reference detected."
---

## Phase 1 — Triage

MUST:
- Check transform property value: `aws iotsitewise get-asset-property-value --asset-id <id> --property-id <transform-prop-id>`
- Verify input property has data: check all referenced measurement properties
- Review formula for syntax errors in model definition
- Check for division by zero or undefined operations

SHOULD:
- Verify all variable names in formula match property logical IDs
- Check if input properties have recent data
- Review model for circular references between transforms

MAY:
- Test formula logic manually with known input values
- Check CloudWatch for transform computation errors

## Phase 2 — Remediate

MUST:
- Ensure all input properties have valid data
- Fix formula syntax errors
- Add guards for division by zero: use if() function

SHOULD:
- Use default values for optional inputs
- Document formula logic and expected behavior

## Output Format

```yaml
root_cause: "transform_error — <specific_cause>"
evidence:
  - type: formula_definition
    content: "<transform formula>"
severity: MEDIUM
mitigation:
  immediate: "Fix formula or ensure input data"
  long_term: "Add defensive formula logic"
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
