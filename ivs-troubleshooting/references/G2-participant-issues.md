# G2 — IVS Real-Time Participant Issues

## Symptoms
- Participant can't publish video/audio
- Subscriber not receiving streams
- Audio/video quality poor in stage

## Triage Commands

```bash
aws ivs-realtime get-stage-session --stage-arn <stage-arn> --session-id <session-id>
aws ivs-realtime list-participants --stage-arn <stage-arn> --session-id <session-id>
aws ivs-realtime get-participant --stage-arn <stage-arn> --session-id <session-id> --participant-id <id>
```

## Common Causes

1. **Token missing PUBLISH capability** — Token only has SUBSCRIBE
2. **Browser permissions** — Camera/microphone not granted
3. **Network restrictions** — Firewall blocking WebRTC (UDP ports)
4. **Participant disconnected** — Network instability
5. **Too many publishers** — Stage publisher limit reached

## Resolution

- Include PUBLISH in token capabilities for publishers
- Ensure browser has camera/microphone permissions
- WebRTC needs UDP connectivity — check firewall rules
- Monitor participant state via `get-participant` API
- For composition: use `aws ivs-realtime start-composition` to combine participants

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete channels during active live streams
- Never reset stream keys without coordinating with the streaming source
- Never modify recording configurations during active recording sessions
- Never delete chat rooms with active participants
- Never change stage configurations during live real-time sessions

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous channel configuration if streaming quality degrades
- Regenerate stream keys and update encoder settings if ingest breaks
- Revert recording configuration changes if auto-recording stops working

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
