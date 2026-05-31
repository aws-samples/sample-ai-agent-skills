# D1 — MediaPackage DRM Configuration Issues

## Symptoms
- DRM-protected content won't play
- License acquisition failures
- Encryption key errors

## Triage Commands

```bash
aws mediapackage describe-origin-endpoint --id <endpoint-id> | jq '.HlsPackage.Encryption // .DashPackage.Encryption // .CmafPackage.Encryption'
# Test SPEKE endpoint (if accessible)
curl -v <speke-key-provider-url>
```

## DRM System IDs

| DRM System | System ID |
|------------|-----------|
| Widevine | edef8ba9-79d6-4ace-a3c8-27dcd51d21ed |
| FairPlay | 94ce86fb-07ff-4f43-adb8-93d2fa968ca2 |
| PlayReady | 9a04f079-9840-4286-ab92-e65be0885f95 |

## Common Causes

1. **SPEKE key provider unreachable** — URL not accessible from MediaPackage
2. **Wrong system IDs** — Must match the DRM system being used
3. **Key provider errors** — Provider returning invalid CPIX response
4. **Encryption mode mismatch** — HLS uses SAMPLE-AES, DASH uses CENC
5. **Resource ID mismatch** — Content ID in request doesn't match provider config

## Resolution

- Verify SPEKE key provider URL is accessible (test with curl from VPC if needed)
- Use correct system IDs for target DRM systems
- Check key provider logs for CPIX request/response errors
- For HLS FairPlay: use SAMPLE-AES encryption method
- For DASH Widevine/PlayReady: use CENC encryption


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
