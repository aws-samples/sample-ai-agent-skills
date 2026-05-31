# C1 — IVS Playback Errors

## Symptoms
- Player shows error or black screen
- Playback URL returns 404
- CORS errors in browser console

## Triage Commands

```bash
aws ivs get-channel --arn <channel-arn> | jq '.channel.playbackUrl'
aws ivs get-stream --channel-arn <channel-arn>
aws cloudwatch get-metric-statistics --namespace AWS/IVS --metric-name ConcurrentViews --dimensions Name=Channel,Value=<channel-arn> --start-time <start> --end-time <end> --period 60 --statistics Maximum
```

## Common Causes

1. **No active stream** — Channel exists but no one is streaming
2. **Wrong playback URL** — Using ingest URL or manually constructed URL
3. **CORS issues** — Browser blocking cross-origin requests
4. **Player SDK version** — Outdated IVS Player SDK
5. **Playback authorization** — Playback auth enabled but no valid token

## Resolution

- Verify stream is active: `aws ivs get-stream --channel-arn <arn>`
- Use playbackUrl from `get-channel` response
- For browser: use IVS Player SDK which handles CORS
- Update to latest IVS Player SDK version
- If playback auth enabled, generate valid playback token server-side

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
