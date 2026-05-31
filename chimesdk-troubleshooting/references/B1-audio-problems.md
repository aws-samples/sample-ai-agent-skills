# B1 — Chime SDK Audio Problems

## Symptoms
- No audio in meeting
- One-way audio
- Audio echo or feedback
- Audio cutting in and out

## Triage Commands

```bash
# Check meeting health via CloudWatch
aws cloudwatch get-metric-statistics --namespace AWS/ChimeSDK --metric-name AttendeeCount --dimensions Name=MeetingId,Value=<meeting-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum
```

## Common Causes

1. **UDP blocked** — Firewall blocking UDP port 3478
2. **Microphone permissions** — Browser/OS not granting mic access
3. **Audio device selection** — Wrong input/output device selected
4. **Echo cancellation** — Missing AEC in client implementation
5. **Network quality** — Insufficient bandwidth for audio (minimum 50 kbps)

## Resolution

- Verify UDP 3478 is open to Chime media endpoints
- Check browser microphone permissions (navigator.mediaDevices.getUserMedia)
- Implement device selection UI using SDK's AudioVideoFacade
- Enable echo cancellation in audio constraints
- Use SDK's meeting health metrics to diagnose quality issues
- Test with Chime SDK meeting readiness checker


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Verify UDP 3478 is open to Chime media endpoints | YELLOW |
| Check browser microphone permissions | GREEN |
| Implement device selection UI using SDK's AudioVideoFacade | GREEN |
| Enable echo cancellation in audio constraints | GREEN |
| Use SDK's meeting health metrics to diagnose quality issues | GREEN |
| Test with Chime SDK meeting readiness checker | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Audio streams contain participant voice data and meeting content

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If firewall rules were modified to open UDP 3478, revert the rules if no longer needed
2. If audio constraints were changed in the client SDK, restore previous audio configuration
3. If device selection was modified, restore default device selection behavior
4. Verify rollback by testing audio in a new meeting session

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
