# C2 — MediaLive HLS Output Issues

## Symptoms
- HLS manifest not generated or incomplete
- Segments missing or variable duration
- Playback stuttering or buffering

## Triage Commands

```bash
# Check HLS output group settings
aws medialive describe-channel --channel-id <id> | jq '.EncoderSettings.OutputGroups[] | select(.OutputGroupSettings.HlsGroupSettings)'

# Check output S3 location
aws s3 ls s3://<bucket>/<path>/ --recursive | tail -20

# Check segment files
aws s3 cp s3://<bucket>/<path>/manifest.m3u8 - | head -20
```

## Common Causes

1. **GOP/segment misalignment** — Segment duration must be multiple of GOP size
2. **Manifest not updating** — S3 eventual consistency or CDN caching stale manifest
3. **Missing renditions** — ABR ladder outputs not all producing segments
4. **Redundant manifest issues** — STANDARD channel manifests from both pipelines
5. **Encryption key delivery** — HLS encryption requires accessible key URL

## Resolution

- Align segment duration with GOP: if GOP=2s, use segment=6s (3 GOPs)
- Set `HlsGroupSettings.ManifestDurationFormat` to `FLOATING_POINT` for accuracy
- For ABR, ensure all renditions have matching segment durations
- Configure `HlsGroupSettings.RedundantManifest` for STANDARD channels
- For encryption, verify key provider URL is accessible to players


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
