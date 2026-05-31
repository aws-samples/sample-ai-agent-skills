---
title: "E2 — Schedule Conflicts"
description: "Diagnose scheduling conflicts"
status: active
severity: MEDIUM
triggers: ["schedule conflict", "overlapping changes", "calendar blocked"]
owner: devops-agent
objective: "Resolve scheduling conflicts"
context: "Multiple changes scheduled at the same time or during Change Calendar restricted periods cause conflicts."
---

## Phase 1 — Triage

MUST:
- Check Change Calendar state for the time window: `aws ssm get-calendar-state --calendar-names <name> --at-time <time>`
- List other changes scheduled for the same window
- Verify no conflicting maintenance windows

SHOULD:
- Check for resource-level conflicts (same resources in multiple changes)

MAY:
- Review change history for the time window

## Phase 2 — Remediate

MUST:
- Reschedule conflicting changes to different windows
- If calendar blocked: use emergency template or wait for open window

SHOULD:
- Implement change scheduling coordination process

MAY:
- Use Change Calendar to define approved change windows

## Common Issues

- symptoms: "Change blocked — calendar in CLOSED state"
  diagnosis: "Change Calendar restricts changes during this period."
  resolution: "Reschedule to an OPEN calendar period or use emergency change template."

## Output Format

```yaml
root_cause: "schedule_conflict — <specific_cause>"
evidence:
  - type: conflict_details
    content: "<conflicting schedules and calendar state>"
severity: MEDIUM
mitigation:
  immediate: "Resolve scheduling conflict"
  long_term: "Implement change scheduling coordination"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Reschedule conflicting changes to different windows | GREEN |
| If calendar blocked: use emergency template or wait for open window | YELLOW |
| Implement change scheduling coordination process | GREEN |
| Use Change Calendar to define approved change windows | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Schedule conflict details may reveal multiple concurrent change activities

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If changes were rescheduled, restore the original schedule if the conflict was resolved
2. If an emergency template was used, document the justification for audit
3. If Change Calendar was modified, restore the previous calendar configuration
4. Verify rollback by confirming `aws ssm get-calendar-state` returns expected state

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
