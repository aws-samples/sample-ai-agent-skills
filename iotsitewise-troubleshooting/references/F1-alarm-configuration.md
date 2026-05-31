---
title: "F1 — Alarm Configuration Issues"
description: "Diagnose SiteWise alarm setup and evaluation problems"
status: active
severity: MEDIUM
triggers:
  - "alarm not triggering"
  - "alarm stuck"
  - "alarm configuration error"
  - "threshold not working"
owner: devops-agent
objective: "Identify and resolve alarm configuration issues"
context: "Alarm issues occur when: threshold misconfigured, source property has no data, alarm model invalid, or notification action fails."
---

## Phase 1 — Triage

MUST:
- Check alarm model definition in asset model
- Verify source property has data and correct type
- Check alarm threshold and comparison operator
- Verify alarm state: check alarm composite model property

SHOULD:
- Check alarm notification action configuration
- Verify IAM role for alarm actions
- Review alarm evaluation frequency

MAY:
- Check alarm state history
- Test with known values that should trigger the alarm

## Phase 2 — Remediate

MUST:
- Fix threshold value and comparison operator
- Ensure source property is receiving data
- Configure alarm notification actions

SHOULD:
- Test alarms with boundary values
- Document alarm thresholds and expected behavior

## Output Format

```yaml
root_cause: "alarm_config — <specific_cause>"
evidence:
  - type: alarm_definition
    content: "<alarm configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix threshold or source property"
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
