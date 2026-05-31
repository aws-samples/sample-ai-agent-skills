# E2 — Chime SDK PSTN Audio Issues

## Symptoms
- PSTN dial-in not working
- Call quality poor on PSTN
- PSTN participant can't hear/be heard

## Triage Commands

```bash
aws chime-sdk-voice list-sip-media-applications --max-results 20
aws chime-sdk-voice list-sip-rules --max-results 20
aws chime-sdk-voice list-phone-numbers --product-type SipMediaApplicationDialIn
```

## Common Causes

1. **Phone number not provisioned** — Number not yet active
2. **SIP rule missing** — No rule routing calls to SIP media app
3. **Lambda not joining meeting** — Lambda must return JoinChimeMeeting action
4. **Meeting expired** — Meeting ended before PSTN participant joined
5. **Audio codec mismatch** — PSTN uses G.711; meeting uses Opus

## Resolution

- Verify phone number is provisioned and assigned
- Verify SIP rule routes to correct SIP media application
- Lambda must return `JoinChimeMeeting` action with valid meeting ID and attendee token
- Create meeting and attendee before PSTN participant calls
- Chime SDK handles codec transcoding automatically — no manual config needed


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Verify phone number is provisioned and assigned | GREEN |
| Verify SIP rule routes to correct SIP media application | GREEN |
| Lambda must return JoinChimeMeeting action with valid meeting ID | YELLOW |
| Create meeting and attendee before PSTN participant calls | GREEN |
| Chime SDK handles codec transcoding automatically | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- PSTN audio involves phone call data and participant voice communications

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If Lambda was modified to change JoinChimeMeeting action, redeploy the previous Lambda version
2. If SIP rule routing was changed, restore the previous routing configuration
3. If phone number assignment was modified, reassign to the previous SIP media application
4. Verify rollback by testing a PSTN dial-in to confirm call routing works correctly

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
