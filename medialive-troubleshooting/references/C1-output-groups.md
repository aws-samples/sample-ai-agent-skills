# C1 — MediaLive Output Group Configuration Issues

## Symptoms
- Output group creation validation errors
- No output produced despite channel running
- Output destination errors

## Triage Commands

```bash
aws medialive describe-channel --channel-id <id> | jq '.EncoderSettings.OutputGroups'
aws medialive describe-channel --channel-id <id> | jq '.Destinations'

# Check S3 destination access
aws s3 ls s3://<output-bucket>/<path>/

# Check MediaPackage channel
aws mediapackage describe-channel --id <mediapackage-channel-id>
```

## Output Group Types

| Type | Destination | Use Case |
|------|-------------|----------|
| HLS_GROUP_SETTINGS | S3, MediaPackage, MediaStore, HTTP | Adaptive streaming |
| DASH_ISO_GROUP_SETTINGS | S3, HTTP | DASH streaming |
| CMAF_INGEST_GROUP_SETTINGS | MediaPackage v2 | CMAF packaging |
| ARCHIVE_GROUP_SETTINGS | S3 | Recording/archive |
| FRAME_CAPTURE_GROUP_SETTINGS | S3 | Thumbnail generation |
| UDP_OUTPUT_SETTINGS | IP:port | Direct UDP output |
| MEDIA_PACKAGE_GROUP_SETTINGS | MediaPackage v1 | Direct to MediaPackage |

## Common Causes

1. **Destination mismatch** — Output group type doesn't match destination type
2. **S3 permissions** — IAM role lacks s3:PutObject on output bucket
3. **MediaPackage channel not found** — Channel ID or ARN incorrect
4. **Missing codec settings** — Output must have video and/or audio codec configuration
5. **Segment duration mismatch** — Must align with GOP size

## Resolution

- Match output group type to destination
- Verify IAM role has write access to S3 output bucket
- For MediaPackage: verify channel exists and has available input endpoints
- Ensure each output has appropriate codec settings configured


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
