---
title: "E2 — Condition Expression Errors"
description: "Diagnose condition expression evaluation failures"
status: active
severity: MEDIUM
triggers:
  - "condition error"
  - "expression failed"
  - "type mismatch"
  - "undefined variable"
owner: devops-agent
objective: "Identify and resolve condition expression errors"
context: "Expression errors occur when: type mismatch in comparison, undefined variable, null input attribute, or syntax error."
---

## Phase 1 — Triage

MUST:
- Review condition expression syntax
- Verify input attribute paths: $input.inputName.attributePath
- Check variable references: $variable.variableName
- Verify comparison types match (string vs number)

SHOULD:
- Check if variables are initialized before use
- Verify logical operator usage (&& for AND, || for OR)
- Check for null/undefined attribute handling

MAY:
- Test expression components individually
- Check if built-in functions are used correctly

## Phase 2 — Remediate

MUST:
- Fix expression syntax to use IoT Events expression language
- Ensure variables are set before referenced
- Match comparison types (don't compare string to number)

SHOULD:
- Add null checks for optional input attributes
- Use convert() function for type conversions

## Output Format

```yaml
root_cause: "condition_error — <specific_cause>"
evidence:
  - type: expression
    content: "<failing expression>"
severity: MEDIUM
mitigation:
  immediate: "Fix expression syntax"
  long_term: "Implement expression testing"
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
- Never delete detector models that are actively monitoring production inputs
- Never modify alarm models during active alarm states without understanding impact
- Never change input routing configurations without verifying downstream detectors
- Never alter state transition logic on production detector models without testing
- Never remove action targets (Lambda, SNS, etc.) from active detector models

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Republish previous detector model version if event processing breaks
- Restore previous alarm model configuration if alarm evaluation fails
- Revert input routing changes if detectors stop receiving data

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
