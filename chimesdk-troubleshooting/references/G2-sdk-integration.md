# G2 — Chime SDK Integration Issues

## Symptoms
- Server-side API integration errors
- Event notification failures
- Media pipeline integration issues

## Triage Commands

```bash
aws chime-sdk-meetings list-tags-for-resource --resource-arn <meeting-arn>
aws events list-rules --name-prefix ChimeSDK
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chime.amazonaws.com --max-results 10
```

## Common Causes

1. **EventBridge not configured** — Meeting events need EventBridge rule
2. **IAM permissions incomplete** — Multiple Chime SDK services need separate permissions
3. **Media pipeline IAM** — Recording/transcription needs additional permissions
4. **Cross-region issues** — Control plane and media plane in different regions
5. **Webhook delivery** — EventBridge target not receiving events

## Resolution

- Set up EventBridge rules for meeting lifecycle events
- IAM policy should include all needed Chime SDK services
- For media pipelines: add `chime-sdk-media-pipelines:*` and S3 write permissions
- Use meeting events: `chime:MeetingStarted`, `chime:MeetingEnded`, `chime:AttendeeJoined`
- Test integration with sample meetings before production


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Set up EventBridge rules for meeting lifecycle events | GREEN |
| IAM policy should include all needed Chime SDK services | YELLOW |
| For media pipelines: add chime-sdk-media-pipelines and S3 write permissions | YELLOW |
| Use meeting events for lifecycle tracking | GREEN |
| Test integration with sample meetings before production | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Media pipeline integrations handle meeting recordings and transcription data

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If EventBridge rules were created, delete them (remove targets first)
2. If IAM permissions were broadened, revert to the previous policy version
3. If media pipeline configurations were changed, restore previous settings
4. If S3 write permissions were added, remove them if no longer needed
5. Verify rollback by confirming EventBridge rules and IAM policies match expected state

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
