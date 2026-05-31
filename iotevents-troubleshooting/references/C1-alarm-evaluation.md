---
title: "C1 — Alarm Evaluation Issues"
description: "Diagnose alarm model evaluation failures"
status: active
severity: HIGH
triggers:
  - "alarm not triggering"
  - "alarm not evaluating"
  - "threshold not detected"
owner: devops-agent
objective: "Identify and resolve alarm evaluation failures"
context: "Alarm evaluation fails when: input not reaching alarm, threshold misconfigured, alarm disabled, or comparison type wrong."
---

## Phase 1 — Triage

MUST:
- Check alarm model: `aws iotevents describe-alarm-model --alarm-model-name <name>`
- Check alarm state: `aws iotevents-data describe-alarm --alarm-model-name <name> --key-value <key>`
- Verify input is being received (check input routing)
- Check threshold value and comparison operator

SHOULD:
- Verify alarm is not in DISABLED state
- Check severity level configuration
- Verify alarm rule expression syntax

MAY:
- List all alarm instances: `aws iotevents-data list-alarms --alarm-model-name <name>`
- Check CloudWatch for alarm evaluation metrics

## Phase 2 — Remediate

MUST:
- Fix threshold value and comparison operator
- Ensure input data is reaching the alarm model
- Enable alarm if in DISABLED state

SHOULD:
- Test with known values that should trigger the alarm
- Configure alarm notification actions

## Output Format

```yaml
root_cause: "alarm_evaluation — <specific_cause>"
evidence:
  - type: alarm_state
    content: "<alarm model and instance state>"
severity: HIGH
mitigation:
  immediate: "Fix threshold or input routing"
  long_term: "Implement alarm testing procedures"
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
