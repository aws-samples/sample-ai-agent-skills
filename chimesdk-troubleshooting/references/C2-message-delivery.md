# C2 — Chime SDK Message Delivery Issues

## Symptoms
- Messages sent but not received
- Message delivery delayed
- Push notifications not working

## Triage Commands

```bash
aws chime-sdk-messaging list-channel-messages --channel-arn <channel-arn> --max-results 10
aws chime-sdk-messaging get-channel-message --channel-arn <channel-arn> --message-id <message-id>
```

## Common Causes

1. **WebSocket not connected** — Client must maintain WebSocket for real-time delivery
2. **Channel membership** — Recipient not a member of the channel
3. **Message processing** — Channel flow processor rejecting messages
4. **Push notification config** — Missing or invalid push notification configuration
5. **Elastic channels** — Different delivery behavior for elastic channels

## Resolution

- Ensure client maintains WebSocket connection via MessagingSession
- Verify recipient is channel member with appropriate permissions
- Check channel flow configuration if messages are being filtered
- For push notifications, configure via app instance with platform credentials
- Use `list-channel-messages` to verify message was persisted


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure client maintains WebSocket connection via MessagingSession | GREEN |
| Verify recipient is channel member with appropriate permissions | GREEN |
| Check channel flow configuration if messages are being filtered | GREEN |
| Configure push notifications via app instance with platform credentials | YELLOW |
| Use list-channel-messages to verify message was persisted | GREEN |

## Escalation Conditions

- Fix affects active meetings/calls

## Data Sensitivity

- **HIGH**: meeting recordings, participant data
- Message content and delivery metadata may contain sensitive communications

## Prohibited Actions

- NEVER suggest disabling encryption on meetings

## Phase 3 — Rollback

1. If channel flow configuration was modified, restore the previous flow processor settings
2. If push notification configuration was changed, revert to previous platform credentials
3. If channel membership was modified, restore previous membership state
4. Verify rollback by sending a test message and confirming delivery

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
