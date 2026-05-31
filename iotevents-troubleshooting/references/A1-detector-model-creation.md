---
title: "A1 — Detector Model Creation Failures"
description: "Diagnose detector model creation and update failures"
status: active
severity: HIGH
triggers:
  - "model creation failed"
  - "cannot create detector"
  - "model validation error"
owner: devops-agent
objective: "Identify and resolve detector model creation failures"
context: "Creation fails when: state machine invalid, IAM role missing, input references wrong, or expression syntax errors."
---

## Phase 1 — Triage

MUST:
- Check model definition for syntax errors
- Verify IAM role exists and has correct trust policy: `aws iam get-role --role-name <role-name>`
- Verify all referenced inputs exist: `aws iotevents list-inputs`
- Check CloudTrail for creation errors

SHOULD:
- Validate condition expressions for syntax
- Verify initial state is defined
- Check for unreachable states in the model

MAY:
- Use IoT Events console visual editor to validate model
- Check account limits for detector models

## Phase 2 — Remediate

MUST:
- Fix state machine definition (states, transitions, actions)
- Create IAM role with iotevents.amazonaws.com trust policy
- Create missing input definitions

SHOULD:
- Test model with simple states before adding complexity
- Document state machine design

## Output Format

```yaml
root_cause: "model_creation — <specific_cause>"
evidence:
  - type: model_definition
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix model definition"
  long_term: "Implement model validation pipeline"
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
