# E1 — IVS Auto-Recording Issues

## Symptoms
- Stream not being recorded
- Recording configuration errors
- Partial or missing recordings

## Triage Commands

```bash
aws ivs list-recording-configurations
aws ivs get-recording-configuration --arn <recording-config-arn>
aws ivs get-channel --arn <channel-arn> | jq '.channel.recordingConfigurationArn'
aws s3 ls s3://<recording-bucket>/ --recursive | tail -20
```

## Common Causes

1. **Recording config not attached** — Must be attached to channel before stream starts
2. **S3 bucket permissions** — IVS needs write access to the bucket
3. **Bucket wrong region** — Must be in same region as IVS channel
4. **Attached during active stream** — Only applies to next stream
5. **Recording config creation failed** — S3 bucket validation failed

## Resolution

- Create recording config: `aws ivs create-recording-configuration --destination-configuration s3={bucketName=<bucket>}`
- Attach to channel: update channel with `--recording-configuration-arn <arn>`
- Ensure S3 bucket policy allows IVS service principal
- Verify bucket is in same region as channel
- Restart stream after attaching recording config

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
