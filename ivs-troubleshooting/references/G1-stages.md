# G1 — IVS Real-Time Stage Issues

## Symptoms
- Stage creation fails
- Participants can't join stage
- Stage session errors

## Triage Commands

```bash
aws ivs-realtime list-stages --max-results 20
aws ivs-realtime get-stage --arn <stage-arn>
aws ivs-realtime list-stage-sessions --stage-arn <stage-arn> --max-results 5
aws ivs-realtime get-stage-session --stage-arn <stage-arn> --session-id <session-id>
```

## Common Causes

1. **Region not supported** — IVS Real-Time available in limited regions
2. **Participant token expired** — Tokens have TTL
3. **Participant limit reached** — Stage has publisher/subscriber limits
4. **Wrong API** — Using ivs instead of ivs-realtime
5. **Browser compatibility** — WebRTC requires modern browser

## Resolution

- Use `aws ivs-realtime` commands (not `aws ivs`)
- Create participant tokens: `aws ivs-realtime create-participant-token --stage-arn <arn>`
- Set appropriate token duration and capabilities (PUBLISH, SUBSCRIBE)
- Check participant limits in service quotas
- Ensure clients use modern browsers with WebRTC support

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
