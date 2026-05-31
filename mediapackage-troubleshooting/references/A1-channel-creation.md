# A1 — MediaPackage Channel Creation Failures

## Symptoms
- `create-channel` returns error
- Channel not appearing in console
- Ingest endpoints not generated

## Triage Commands

```bash
aws mediapackage list-channels --max-results 20
aws mediapackage describe-channel --id <channel-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateChannel --max-results 5
aws service-quotas get-service-quota --service-code mediapackage --quota-code L-352B8AF1
```

## Common Causes

1. **Channel ID already exists** — Channel IDs must be unique per region
2. **Service quota reached** — Default limit on channels per region
3. **IAM permissions** — Caller needs mediapackage:CreateChannel permission
4. **Invalid channel ID** — Must match pattern: lowercase, numbers, hyphens

## Resolution

- Use unique channel IDs per region
- Check quotas: `aws service-quotas list-service-quotas --service-code mediapackage`
- Verify IAM permissions for the calling principal
- After creation, note the two ingest endpoint URLs and credentials


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

- **NEVER** delete an origin endpoint that is actively serving live or VOD content to viewers
- **NEVER** modify DRM key rotation settings during an active live stream
- **NEVER** remove CDN authorization headers from endpoints serving production traffic
- **NEVER** change packaging configurations on live channels without a tested rollback plan
- **NEVER** delete harvest jobs or live-to-VOD assets without confirming content retention requirements

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
