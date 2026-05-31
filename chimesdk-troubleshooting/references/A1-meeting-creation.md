# A1 — Chime SDK Meeting Creation Failures

## Symptoms
- CreateMeeting API returns error
- Meeting not accessible after creation
- Region-related errors

## Triage Commands

```bash
aws chime-sdk-meetings create-meeting --client-request-token <token> --media-region <region> --external-meeting-id <ext-id>
aws chime-sdk-meetings get-meeting --meeting-id <meeting-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateMeeting --max-results 5
```

## Common Causes

1. **Invalid media region** — Not all AWS regions are valid media regions
2. **IAM permissions** — Caller needs chime-sdk-meetings:CreateMeeting
3. **Service quota** — Concurrent meeting limit reached
4. **Duplicate client request token** — Token must be unique per request
5. **Wrong API namespace** — Using legacy chime instead of chime-sdk-meetings

## Resolution

- Use valid media region (us-east-1, us-west-2, eu-west-1, ap-southeast-1, etc.)
- Verify IAM permissions for chime-sdk-meetings service
- Use unique ClientRequestToken (UUID recommended)
- Use `aws chime-sdk-meetings` namespace, not `aws chime`


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use valid media region | GREEN |
| Verify IAM permissions for chime-sdk-meetings service | YELLOW |
| Use unique ClientRequestToken (UUID recommended) | GREEN |
| Use aws chime-sdk-meetings namespace, not aws chime | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Meeting creation parameters may contain external meeting IDs linked to business processes

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If a meeting was created in error, delete it: `aws chime-sdk-meetings delete-meeting --meeting-id <id>`
2. If IAM permissions were broadened, revert to the previous policy version
3. If the media region was changed, create a new meeting in the correct region and migrate attendees
4. Verify rollback by confirming `aws chime-sdk-meetings get-meeting` returns expected state

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
