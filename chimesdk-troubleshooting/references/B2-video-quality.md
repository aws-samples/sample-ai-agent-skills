# B2 — Chime SDK Video Quality Issues

## Symptoms
- Video freezing or pixelated
- Video not displaying
- Low resolution despite high-quality camera
- Video bandwidth issues

## Triage Commands

```bash
aws cloudwatch get-metric-statistics --namespace AWS/ChimeSDK --metric-name MeetingDuration --dimensions Name=MeetingId,Value=<meeting-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum
```

## Common Causes

1. **Bandwidth insufficient** — Video needs 300 kbps-2.5 Mbps depending on quality
2. **Camera permissions** — Browser/OS not granting camera access
3. **Simulcast not enabled** — Without simulcast, all viewers get same quality
4. **Too many video tiles** — CPU/bandwidth overload with many participants
5. **TURN relay** — UDP blocked, falling back to TURN adds latency

## Resolution

- Enable simulcast for adaptive quality: configure in MeetingSessionConfiguration
- Implement video tile management — limit active video tiles (e.g., 16 max)
- Use priority-based video subscription for large meetings
- Check camera permissions and device selection
- Monitor bandwidth using SDK's connection health data
- For content sharing: use separate content share API, not video


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Enable simulcast for adaptive quality | GREEN |
| Implement video tile management — limit active video tiles | GREEN |
| Use priority-based video subscription for large meetings | GREEN |
| Check camera permissions and device selection | GREEN |
| Monitor bandwidth using SDK's connection health data | GREEN |
| For content sharing: use separate content share API | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Video streams contain participant visual data and shared content

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If simulcast was enabled, disable it if it causes issues in the MeetingSessionConfiguration
2. If video tile limits were changed, restore previous tile management settings
3. If priority-based subscription was configured, revert to default subscription behavior
4. Verify rollback by testing video quality in a new meeting session

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
