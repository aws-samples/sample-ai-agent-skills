# G1 — Chime SDK Client SDK Issues

## Symptoms
- SDK initialization fails
- Can't connect to meeting
- SDK version compatibility errors

## Triage Commands

```bash
# Server-side verification
aws chime-sdk-meetings get-meeting --meeting-id <meeting-id>
aws chime-sdk-meetings get-attendee --meeting-id <meeting-id> --attendee-id <attendee-id>
```

## SDK Platforms

| Platform | Package |
|----------|---------|
| JavaScript | amazon-chime-sdk-js |
| React | amazon-chime-sdk-component-library-react |
| iOS | AmazonChimeSDK (CocoaPods/SPM) |
| Android | amazon-chime-sdk (Maven) |

## Common Causes

1. **Wrong SDK version** — Incompatible client/server versions
2. **Missing JoinInfo** — Meeting and attendee info not passed correctly
3. **Browser compatibility** — WebRTC not supported in older browsers
4. **HTTPS required** — getUserMedia requires secure context
5. **Bundle size** — SDK not properly tree-shaken

## Resolution

- Use latest compatible SDK version
- Pass complete MeetingSessionConfiguration (meeting + attendee response)
- Test in Chrome, Firefox, Safari, Edge (latest versions)
- Serve application over HTTPS (required for camera/mic access)
- Follow SDK migration guides when upgrading versions


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use latest compatible SDK version | GREEN |
| Pass complete MeetingSessionConfiguration | GREEN |
| Test in Chrome, Firefox, Safari, Edge (latest versions) | GREEN |
| Serve application over HTTPS | GREEN |
| Follow SDK migration guides when upgrading versions | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Client SDK handles meeting session tokens and media streams

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If SDK version was upgraded, revert to the previous SDK version in package dependencies
2. If MeetingSessionConfiguration was changed, restore previous configuration
3. If HTTPS configuration was modified, ensure the application still serves over HTTPS
4. Verify rollback by testing meeting join flow in supported browsers

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
