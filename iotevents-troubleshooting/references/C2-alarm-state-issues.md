---
title: "C2 — Alarm State Issues"
description: "Diagnose alarm state transition problems"
status: active
severity: MEDIUM
triggers:
  - "alarm stuck"
  - "alarm won't clear"
  - "alarm state wrong"
  - "acknowledge failed"
owner: devops-agent
objective: "Identify and resolve alarm state transition issues"
context: "State issues occur when: acknowledge flow misconfigured, alarm latched, snooze active, or state transition rules misunderstood."
---

## Phase 1 — Triage

MUST:
- Check current alarm state: `aws iotevents-data describe-alarm --alarm-model-name <name> --key-value <key>`
- Review alarm model acknowledge flow configuration
- Check if alarm is latched (requires explicit acknowledge)
- Verify snooze duration if in SNOOZE_DISABLED state

SHOULD:
- Review alarm state transition history
- Check if acknowledge action is configured
- Verify IAM permissions for alarm state changes

MAY:
- Manually acknowledge alarm: `aws iotevents-data batch-acknowledge-alarm --acknowledge-action-requests '[{"alarmModelName":"<name>","keyValue":"<key>","requestId":"test"}]'`

## Phase 2 — Remediate

MUST:
- Acknowledge alarm if latched and condition cleared
- Disable snooze if blocking state transitions
- Fix acknowledge flow configuration

SHOULD:
- Document alarm state machine behavior for operators
- Configure appropriate acknowledge requirements

## Output Format

```yaml
root_cause: "alarm_state — <specific_cause>"
evidence:
  - type: alarm_instance
    content: "<alarm state details>"
severity: MEDIUM
mitigation:
  immediate: "Fix alarm state or acknowledge"
  long_term: "Document alarm workflows"
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
