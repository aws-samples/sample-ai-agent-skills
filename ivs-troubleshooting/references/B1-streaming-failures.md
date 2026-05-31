# B1 — IVS Streaming Failures

## Symptoms
- Stream won't start
- Stream disconnects frequently
- Stream health degraded

## Triage Commands

```bash
aws ivs get-stream --channel-arn <channel-arn>
aws ivs list-stream-sessions --channel-arn <channel-arn> --max-results 5
aws ivs get-stream-session --channel-arn <channel-arn> --stream-id <stream-id>
aws cloudwatch get-metric-statistics --namespace AWS/IVS --metric-name IngestFramerate --dimensions Name=Channel,Value=<channel-arn> --start-time <start> --end-time <end> --period 60 --statistics Average
```

## Common Causes

1. **Bitrate exceeds channel limit** — BASIC: 1.5 Mbps, STANDARD: 8.5 Mbps
2. **Wrong protocol** — Must use RTMPS, not RTMP
3. **Invalid stream key** — Deleted or wrong key
4. **Encoder settings incompatible** — Unsupported codec or resolution
5. **Network instability** — Encoder's upload bandwidth insufficient

## Resolution

- Check channel type limits and match encoder bitrate accordingly
- Verify RTMPS protocol in encoder settings
- Check stream session events: `get-stream-session` shows truncation events
- Use H.264 video codec and AAC audio codec
- Ensure stable upload bandwidth of at least 1.5x target bitrate

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
