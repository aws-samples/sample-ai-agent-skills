# B2 — IVS Ingest Issues

## Symptoms
- Ingest connection rejected
- Poor ingest quality metrics
- Ingest bitrate fluctuating

## Triage Commands

```bash
aws ivs get-stream-session --channel-arn <channel-arn> --stream-id <stream-id>
aws cloudwatch get-metric-statistics --namespace AWS/IVS --metric-name IngestVideoBitrate --dimensions Name=Channel,Value=<channel-arn> --start-time <start> --end-time <end> --period 60 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/IVS --metric-name IngestAudioBitrate --dimensions Name=Channel,Value=<channel-arn> --start-time <start> --end-time <end> --period 60 --statistics Average
```

## Recommended Ingest Settings

| Setting | Recommended |
|---------|-------------|
| Protocol | RTMPS |
| Video Codec | H.264 (AVC) |
| Audio Codec | AAC |
| Keyframe Interval | 2 seconds |
| Resolution | Up to 1080p (STANDARD) |
| Framerate | 30 or 60 fps |

## Resolution

- Use H.264 Baseline/Main/High profile
- Set keyframe interval to 2 seconds for optimal ABR
- Use AAC-LC audio at 128 kbps stereo
- Ensure encoder connects to nearest IVS ingest endpoint
- Monitor IngestVideoBitrate for stability

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
