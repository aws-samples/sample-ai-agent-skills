# E2 — MediaLive Health Monitoring

## Symptoms
- Need to set up monitoring for MediaLive channels
- Missing visibility into channel health
- No alerting on failures

## Triage Commands

```bash
# Key CloudWatch metrics
aws cloudwatch list-metrics --namespace AWS/MediaLive --dimensions Name=ChannelId,Value=<id>

# Network throughput
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=ChannelId,Value=<id>,Name=Pipeline,Value=PIPELINE_0 --start-time <start> --end-time <end> --period 60 --statistics Average

# Output metrics
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name Output4xxErrors --dimensions Name=ChannelId,Value=<id>,Name=Pipeline,Value=PIPELINE_0 --start-time <start> --end-time <end> --period 60 --statistics Sum
```

## Key Metrics to Monitor

| Metric | Alert Threshold | Meaning |
|--------|----------------|---------|
| ActiveAlerts | > 0 | Pipeline has active alerts |
| InputLossSeconds | > 0 | Input signal lost |
| NetworkIn | Drop > 50% | Input throughput degraded |
| Output4xxErrors | > 0 | Output destination errors |
| FillMsec | > 0 | Input loss fill active |

## Resolution

- Set up CloudWatch alarms on ActiveAlerts > 0 for each pipeline
- Monitor NetworkIn for input throughput drops
- Use EventBridge for MediaLive state change notifications
- Configure SNS topics for alert notifications
- Dashboard: combine ActiveAlerts, NetworkIn, Output4xxErrors per channel


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
