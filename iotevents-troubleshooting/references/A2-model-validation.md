---
title: "A2 — Model Validation Errors"
description: "Diagnose detector model validation and expression errors"
status: active
severity: MEDIUM
triggers:
  - "validation error"
  - "expression invalid"
  - "condition syntax error"
owner: devops-agent
objective: "Identify and resolve model validation errors"
context: "Validation errors occur when: expressions have syntax errors, variable types mismatch, input paths invalid, or timer durations below minimum."
---

## Phase 1 — Triage

MUST:
- Review all condition expressions for syntax
- Verify input attribute paths match input definitions
- Check variable assignments for type consistency
- Verify timer durations are >= 60 seconds

SHOULD:
- Check action configurations for required parameters
- Verify SNS topic ARNs, SQS queue URLs, Lambda function ARNs
- Validate setVariable expressions

MAY:
- Test expressions with sample data manually
- Check for unused variables or states

## Phase 2 — Remediate

MUST:
- Fix expression syntax: use $input.name.path for inputs, $variable.name for variables
- Correct timer durations to >= 60 seconds
- Fix action target ARNs

SHOULD:
- Use consistent variable naming
- Add comments to complex expressions

## Output Format

```yaml
root_cause: "model_validation — <specific_cause>"
evidence:
  - type: validation_error
    content: "<specific validation message>"
severity: MEDIUM
mitigation:
  immediate: "Fix expression or configuration"
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
