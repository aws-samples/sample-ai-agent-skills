---
title: "E1 — State Transition Logic Issues"
description: "Diagnose state transition evaluation problems"
status: active
severity: MEDIUM
triggers:
  - "transition not firing"
  - "wrong state"
  - "stuck in state"
  - "transition order"
owner: devops-agent
objective: "Identify and resolve state transition logic issues"
context: "Transition issues occur when: condition never true, evaluation order wrong, input not matching, or transition target state invalid."
---

## Phase 1 — Triage

MUST:
- Check current detector state: `aws iotevents-data describe-detector --detector-model-name <name> --key-value <key>`
- Review transition conditions in the current state
- Verify input data satisfies transition condition
- Check transition evaluation order (first match wins)

SHOULD:
- Verify condition expression syntax
- Check if multiple transitions compete (order matters)
- Verify target state exists in model

MAY:
- Trace through state machine logic with sample data
- Check if onInput events modify variables that affect transitions

## Phase 2 — Remediate

MUST:
- Fix transition condition expressions
- Reorder transitions if priority is wrong (first match wins)
- Ensure input data contains required attributes

SHOULD:
- Document state machine logic and expected transitions
- Test transitions with boundary values

## Output Format

```yaml
root_cause: "transition_logic — <specific_cause>"
evidence:
  - type: detector_state
    content: "<current state and conditions>"
severity: MEDIUM
mitigation:
  immediate: "Fix condition expressions"
  long_term: "Document and test state machine logic"
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
