# D2 — IVS Chat Messaging Issues

## Symptoms
- Messages not sending or receiving
- Chat token errors
- Message moderation issues

## Triage Commands

```bash
aws ivschat get-room --identifier <room-arn>
aws ivschat list-rooms --max-results 20
```

## Common Causes

1. **Expired chat token** — Tokens have configurable TTL (1-180 minutes)
2. **Token generated client-side** — Must be generated server-side with AWS credentials
3. **Wrong room ARN in token** — Token must match the room being connected to
4. **Rate limiting** — Message rate limits per user/room
5. **Moderation rules** — Message rejected by auto-moderation

## Resolution

- Generate tokens server-side: `aws ivschat create-chat-token --room-identifier <arn> --user-id <user-id>`
- Set appropriate `sessionDurationInMinutes` (default 60)
- Include `capabilities` in token: SEND_MESSAGE, DISCONNECT_USER, DELETE_MESSAGE
- For moderation, configure message review handler (Lambda function)
- Handle token expiration in client — regenerate before expiry

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
