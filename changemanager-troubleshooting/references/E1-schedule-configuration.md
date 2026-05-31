---
title: "E1 — Schedule Configuration"
description: "Diagnose scheduled change configuration issues"
status: active
severity: MEDIUM
triggers: ["schedule error", "wrong time", "schedule not working"]
owner: devops-agent
objective: "Resolve scheduled change configuration"
context: "Changes can be scheduled for specific times. Schedule requires approval completion before execution time."
---

## Phase 1 — Triage

MUST:
- Check scheduled time in change request
- Verify all approvals are complete before scheduled time
- Check Change Calendar for the scheduled window

SHOULD:
- Verify timezone configuration
- Check if schedule conflicts with maintenance windows

MAY:
- Review past scheduled changes for patterns

## Phase 2 — Remediate

MUST:
- Ensure approvals complete before scheduled time
- Verify Change Calendar allows execution at scheduled time

SHOULD:
- Schedule with buffer time for approval completion
- Coordinate with approvers before scheduling

MAY:
- Use recurring schedules for routine changes

## Common Issues

- symptoms: "Scheduled change didn't execute at specified time"
  diagnosis: "Approvals not complete or Change Calendar blocked."
  resolution: "Complete approvals before schedule. Check calendar state."

## Output Format

```yaml
root_cause: "schedule_configuration — <specific_cause>"
evidence:
  - type: schedule_details
    content: "<schedule time, approval status, calendar state>"
severity: MEDIUM
mitigation:
  immediate: "Fix scheduling issue"
  long_term: "Coordinate approvals with scheduling"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure approvals complete before scheduled time | GREEN |
| Verify Change Calendar allows execution at scheduled time | GREEN |
| Schedule with buffer time for approval completion | GREEN |
| Coordinate with approvers before scheduling | GREEN |
| Use recurring schedules for routine changes | YELLOW |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Schedule configurations may reveal maintenance windows and operational patterns

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If a scheduled change was configured incorrectly, cancel it before execution time
2. If the schedule was modified, restore the previous schedule configuration
3. If recurring schedules were set up, disable them if no longer appropriate
4. Verify rollback by confirming the change request schedule matches expected state

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
