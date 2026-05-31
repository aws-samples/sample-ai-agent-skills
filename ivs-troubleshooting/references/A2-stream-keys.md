# A2 — IVS Stream Key Issues

## Symptoms
- Encoder can't authenticate to IVS
- Stream key not working
- Need to rotate stream key

## Triage Commands

```bash
aws ivs list-stream-keys --channel-arn <channel-arn>
aws ivs get-stream-key --arn <stream-key-arn>
```

## Common Causes

1. **Stream key deleted** — Key was deleted and encoder still using old key
2. **Wrong channel** — Stream key from different channel
3. **Key exposed** — Compromised key needs rotation
4. **Encoder using RTMP** — Must use RTMPS with stream key

## Resolution

- List keys: `aws ivs list-stream-keys --channel-arn <arn>`
- Create new key: `aws ivs create-stream-key --channel-arn <arn>`
- Delete compromised key: `aws ivs delete-stream-key --arn <key-arn>`
- Update encoder with new stream key and RTMPS ingest endpoint
- Stream key value is only shown at creation — store securely

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
