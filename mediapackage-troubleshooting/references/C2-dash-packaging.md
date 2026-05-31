# C2 — MediaPackage DASH Packaging Issues

## Symptoms
- DASH MPD manifest errors
- Player fails to parse manifest
- Missing periods or adaptation sets

## Triage Commands

```bash
aws mediapackage describe-origin-endpoint --id <dash-endpoint-id> | jq '.DashPackage'
curl -s "https://<endpoint-url>/index.mpd" | head -50
```

## Common Causes

1. **ManifestLayout** — FULL vs COMPACT affects player compatibility
2. **Profile** — NONE vs HBBTV_1_5 affects manifest structure
3. **UTC timing** — Missing or wrong UTC timing server URL
4. **Period triggers** — SCTE-35 ad markers creating unexpected periods
5. **Suggested presentation delay** — Too low causes buffering

## Resolution

- Use `ManifestLayout: COMPACT` for most players
- Set `Profile: NONE` for broadest compatibility
- Configure `UtcTiming: HTTP-ISO` with reliable timing server
- Set `SuggestedPresentationDelaySeconds` to 3x segment duration
- For multi-period, verify SCTE-35 markers are correctly configured


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
