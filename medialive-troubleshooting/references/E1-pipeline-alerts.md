# E1 — MediaLive Pipeline Alerts

## Symptoms
- Pipeline alert notifications
- ActiveAlerts metric increasing
- Channel running but output degraded

## Triage Commands

```bash
# Check pipeline details and alerts
aws medialive describe-channel --channel-id <id> | jq '.PipelineDetails'

# Active alerts metric
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name ActiveAlerts --dimensions Name=ChannelId,Value=<id>,Name=Pipeline,Value=PIPELINE_0 --start-time <start> --end-time <end> --period 60 --statistics Maximum

# Input loss metric
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name InputLossSeconds --dimensions Name=ChannelId,Value=<id>,Name=Pipeline,Value=PIPELINE_0 --start-time <start> --end-time <end> --period 60 --statistics Sum
```

## Common Alert Types

| Alert | Meaning |
|-------|---------|
| INPUT_LOSS | No input data received |
| OUTPUT_LOSS | Cannot write to output destination |
| PIPELINE_IDLE | Pipeline not processing |
| INPUT_VIDEO_LOSS | Video track lost from input |
| INPUT_AUDIO_LOSS | Audio track lost from input |

## Resolution

- INPUT_LOSS: verify source encoder is sending, check security groups for PUSH inputs
- OUTPUT_LOSS: verify S3/MediaPackage destination is accessible, check IAM permissions
- PIPELINE_IDLE: check if channel is in RECOVERING state
- Configure CloudWatch alarms on ActiveAlerts for proactive monitoring
- For STANDARD channels, check both PIPELINE_0 and PIPELINE_1 independently


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** stop a live channel during an active broadcast without explicit customer authorization
- **NEVER** modify input security groups to remove IP allowlisting on production channels
- **NEVER** delete input attachments on running channels without a verified failover path
- **NEVER** change encoding profiles on a live channel without scheduling a maintenance window
- **NEVER** disable pipeline redundancy on channels serving production live events

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
