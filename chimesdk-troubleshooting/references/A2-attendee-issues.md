# A2 — Chime SDK Attendee Issues

## Symptoms
- CreateAttendee fails
- Attendee can't join meeting
- Attendee token invalid

## Triage Commands

```bash
aws chime-sdk-meetings list-attendees --meeting-id <meeting-id>
aws chime-sdk-meetings get-attendee --meeting-id <meeting-id> --attendee-id <attendee-id>
```

## Common Causes

1. **Meeting expired** — Meeting ended before attendee creation
2. **Attendee limit** — Maximum 250 attendees per meeting
3. **Duplicate ExternalUserId** — Must be unique within a meeting
4. **Token passed insecurely** — JoinInfo must be transmitted securely to client
5. **Wrong meeting ID** — Attendee created for different meeting

## Resolution

- Verify meeting is still active: `aws chime-sdk-meetings get-meeting --meeting-id <id>`
- Use unique ExternalUserId per attendee
- Pass JoinInfo (meeting + attendee) to client SDK securely (HTTPS)
- For batch creation: `aws chime-sdk-meetings batch-create-attendee`
- Client uses JoinInfo to initialize MeetingSession in SDK


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Verify meeting is still active | GREEN |
| Use unique ExternalUserId per attendee | GREEN |
| Pass JoinInfo to client SDK securely (HTTPS) | GREEN |
| For batch creation: use batch-create-attendee | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Attendee JoinInfo tokens provide meeting access and must be transmitted securely

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If attendees were created in error, they will be cleaned up when the meeting ends
2. If JoinInfo was exposed insecurely, delete the meeting and create a new one with new attendees
3. If ExternalUserId conflicts occurred, delete the attendee and recreate with a unique ID
4. Verify rollback by confirming `aws chime-sdk-meetings list-attendees` shows expected attendees

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
