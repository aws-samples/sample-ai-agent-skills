# Z1 — IVS General Troubleshooting

## When No Specific Runbook Matches

## General Diagnostic Commands

```bash
aws ivs list-channels --max-results 20
aws ivs get-channel --arn <channel-arn>
aws ivs list-streams --filter-by health=STREAM_LIVE
aws ivs list-stream-sessions --channel-arn <channel-arn> --max-results 10
aws ivschat list-rooms --max-results 20
aws ivs-realtime list-stages --max-results 20
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ivs.amazonaws.com --max-results 20
aws service-quotas list-service-quotas --service-code ivs
```

## Escalation Checklist

1. Channel ARN and region
2. Channel type (BASIC/STANDARD/ADVANCED)
3. Stream session ID and health status
4. Ingest configuration (codec, bitrate, resolution)
5. Player SDK version and platform
6. CloudWatch metrics (IngestVideoBitrate, ConcurrentViews, LiveLatency)
7. Error messages from stream session events
8. Timestamp of when issue started

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
