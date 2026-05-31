---
title: "E1 — Change Calendar Integration"
description: "Diagnose Change Calendar blocking SSM Automation executions and calendar state issues"
status: active
severity: MEDIUM
triggers: ["calendar blocked", "CalendarStateIsClosedException", "change freeze", "calendar state", "automation blocked by calendar"]
owner: devops-agent
objective: "Resolve Change Calendar integration issues blocking automation executions"
context: "Change Calendar controls when automations can run. When CLOSED, executions referencing the calendar are blocked immediately — not queued or delayed. Calendar state is checked at execution start time only."
---

## Phase 1 — Triage

MUST:
- Check calendar state: `aws ssm get-calendar-state --calendar-names <calendar-name>`
- Verify which calendars the automation references in its document
- Check execution failure message for CalendarStateIsClosedException

SHOULD:
- List all calendars: `aws ssm list-documents --document-filter-list key=DocumentType,value=ChangeCalendar`
- Review calendar schedule to determine next OPEN window
- Check if the calendar has exceptions configured

MAY:
- Review CloudTrail for calendar state changes
- Check if other automations are also blocked by the same calendar

## Phase 2 — Remediate

MUST:
- Wait for calendar OPEN window and retry execution
- Or add a calendar exception for emergency changes (with approval)
- Verify the correct calendar is referenced in the automation document

SHOULD:
- Schedule automations to run during known OPEN windows
- Configure monitoring for calendar state changes
- Document the calendar schedule for the operations team

MAY:
- Implement a pre-check step that verifies calendar state before critical operations
- Set up EventBridge rules to trigger automations when calendar opens

## Safety Ratings

```
safety_ratings:
  - "get-calendar-state: GREEN — read-only calendar state check"
  - "list-documents (ChangeCalendar): GREEN — read-only calendar listing"
  - "update-document (calendar): RED — modifies change management controls"
  - "create-document (calendar exception): YELLOW — adds exception to change freeze"
```

## Escalation Conditions

- Emergency change required during a change freeze window
- Calendar misconfiguration blocks all automations unexpectedly
- Calendar state does not match the expected schedule
- Multiple teams affected by calendar blocking

## Data Sensitivity

- `aws ssm get-calendar-state`: LOW — returns OPEN or CLOSED state
- `aws ssm list-documents`: LOW — returns document names
- `aws ssm get-document (calendar)`: LOW — returns calendar schedule definition
- `aws cloudtrail lookup-events`: MEDIUM — reveals who modified calendar

## Prohibited Actions

- NEVER modify a change calendar to bypass change management without emergency approval
- NEVER delete a change calendar to unblock automations
- NEVER add permanent exceptions that effectively disable the calendar
- NEVER ignore calendar blocks — they exist for change management governance

## Phase 3 — Rollback

1. If a calendar exception was added incorrectly, remove it by updating the calendar document
2. If the calendar was modified, restore from version control or previous version
3. Verify calendar state returns to expected schedule after changes
4. Notify change management team of any calendar modifications
5. Document the emergency change and calendar exception for audit

## Common Issues

- symptoms: "Automation fails with CalendarStateIsClosedException"
  diagnosis: "The referenced Change Calendar is in CLOSED state."
  resolution: "Wait for the calendar to open, or add an approved exception for emergency changes."

- symptoms: "Calendar shows OPEN but automation still blocked"
  diagnosis: "The automation references multiple calendars, and one of them is CLOSED."
  resolution: "Check all calendars referenced in the document. All must be OPEN for execution to proceed."

- symptoms: "Calendar state does not match expected schedule"
  diagnosis: "The calendar document has incorrect schedule definition or timezone issues."
  resolution: "Review the calendar document content. Verify timezone settings and schedule entries."

## Output Format

```yaml
root_cause: "change_calendar — <specific_cause>"
evidence:
  - type: calendar_state
    content: "<calendar state and schedule details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve calendar blocking"
  long_term: "Align automation schedules with calendar windows"
```

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
