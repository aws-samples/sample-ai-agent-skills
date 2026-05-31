---
title: "C3 — Composite Alarms"
description: "Diagnose composite alarm evaluation and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "composite alarm"
  - "alarm rule"
  - "alarm expression"
  - "parent alarm"
  - "combined alarms"
owner: devops-agent
objective: "Identify why a composite alarm is not evaluating correctly or not triggering actions"
context: "Composite alarms use a rule expression to combine the states of other alarms using AND, OR, NOT, TRUE, FALSE operators. They do NOT inherit actions from child alarms — actions must be configured separately. Composite alarms can suppress child alarm actions using ActionsSuppressor."
---

## Phase 1 — Triage

MUST:
- Check composite alarm configuration: `aws cloudwatch describe-alarms --alarm-names <name> --alarm-types CompositeAlarm`
- Review the AlarmRule expression for correctness
- Verify all child alarms referenced in the rule exist
- Check if alarm actions are configured on the composite alarm itself

SHOULD:
- Check child alarm states: `aws cloudwatch describe-alarms --alarm-names <child1> <child2>`
- Verify ActionsSuppressor configuration if child alarm actions are being suppressed
- Check alarm history for state transitions

MAY:
- Test the rule expression logic manually with current child alarm states
- Check if the composite alarm has insufficient data actions configured

## Phase 2 — Remediate

MUST:
- Fix the AlarmRule expression syntax
- Add alarm actions directly to the composite alarm (they are not inherited)
- Ensure all referenced child alarms exist and are not deleted

SHOULD:
- Configure ActionsSuppressor to prevent notification storms during deployments
- Set up OK actions and insufficient data actions on the composite alarm
- Test the composite alarm by manually setting a child alarm state

MAY:
- Document the composite alarm logic for the operations team
- Create a dashboard showing composite and child alarm states together

## Common Issues

- symptoms: "Composite alarm in ALARM but no notification sent"
  diagnosis: "Composite alarm has no alarm actions configured. Actions are not inherited from child alarms."
  resolution: "Add SNS topic or other actions directly to the composite alarm."

- symptoms: "Composite alarm not entering ALARM when expected"
  diagnosis: "AlarmRule expression logic is incorrect (AND vs OR)."
  resolution: "Review the rule: AND requires ALL children in ALARM, OR requires ANY."

- symptoms: "Child alarm actions not firing when composite alarm exists"
  diagnosis: "ActionsSuppressor is configured on the composite alarm, suppressing child actions."
  resolution: "Review ActionsSuppressor settings. Remove or adjust the suppressor wait period."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix AlarmRule expression | YELLOW | Changes composite alarm evaluation logic |
| Add alarm actions to composite alarm | GREEN | Additive notification configuration |
| Adjust ActionsSuppressor | YELLOW | Changes child alarm action suppression behavior |

## Escalation Conditions

- Composite alarm not triggering for production incidents — escalate to operations team
- ActionsSuppressor misconfigured, suppressing critical notifications — escalate to alarm owner
- Composite alarm logic changes — escalate to monitoring team for review

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch describe-alarms` (CompositeAlarm) | MEDIUM | Exposes alarm rule and actions |
| `aws cloudwatch describe-alarms` (child alarms) | MEDIUM | Exposes child alarm states |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** modify composite alarm rules without understanding the full alarm hierarchy

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed AlarmRule expression | Restore previous expression |
| Added alarm actions | Remove added actions |
| Adjusted ActionsSuppressor | Restore previous suppressor configuration |

## Output Format

```yaml
root_cause: "composite_alarm — <specific_cause>"
evidence:
  - type: composite_alarm_config
    content: "<alarm rule and actions>"
  - type: child_alarm_states
    content: "<child alarm current states>"
severity: MEDIUM
mitigation:
  immediate: "Fix alarm rule, add actions, or adjust ActionsSuppressor"
  long_term: "Document composite alarm logic and test during change windows"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
