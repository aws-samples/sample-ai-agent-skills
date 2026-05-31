# A1 — IVS Channel Creation Failures

## Symptoms
- `create-channel` returns error
- Channel not appearing after creation
- Invalid parameter errors

## Triage Commands

```bash
aws ivs list-channels --max-results 20
aws ivs get-channel --arn <channel-arn>
aws service-quotas get-service-quota --service-code ivs --quota-code L-4733B942
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateChannel --max-results 5
```

## Common Causes

1. **Region not supported** — IVS not available in all regions
2. **Service quota reached** — Channel limit per account/region
3. **Invalid channel type** — Must be BASIC, STANDARD, or ADVANCED_SD/ADVANCED_HD
4. **IAM permissions** — Caller needs ivs:CreateChannel
5. **Recording config invalid** — Referenced recording configuration doesn't exist

## Resolution

- Verify IVS is available in target region
- Check channel quotas: `aws service-quotas list-service-quotas --service-code ivs`
- Use valid channel type for your use case
- If attaching recording config, verify it exists first

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
