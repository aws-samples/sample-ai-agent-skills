---
title: "G1 — Content Protection (DRM) Issues"
description: "Diagnose DRM and content protection configuration problems"
status: active
severity: HIGH
triggers:
  - "DRM"
  - "content protection"
  - "SPEKE"
  - "Widevine"
  - "FairPlay"
  - "PlayReady"
owner: devops-agent
objective: "Resolve DRM content protection configuration and key exchange issues"
context: "MediaConvert uses SPEKE (Secure Packager and Encoder Key Exchange) protocol for DRM. It requires an external key provider. Supported DRM systems include Widevine, FairPlay, and PlayReady depending on the output format."
---

## Phase 1 — Triage

MUST:
- Check job DRM settings: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Verify SPEKE key provider URL is accessible
- Check job error messages for DRM-specific errors
- Verify output group supports the configured DRM system

SHOULD:
- Test SPEKE key provider endpoint independently
- Check IAM role has permission to call the key provider (if API Gateway)
- Verify DRM system compatibility with output format (FairPlay=HLS, Widevine=DASH)

MAY:
- Check key provider logs for request/response details
- Verify content key ID configuration

## Phase 2 — Remediate

MUST:
- If key provider unreachable: verify URL and network access
- If DRM/format mismatch: use compatible DRM for the output group
- If key exchange fails: check SPEKE key provider configuration and logs

SHOULD:
- Use CMAF output for multi-DRM (Widevine + FairPlay) with single encoding
- Test DRM configuration with a short clip before full content
- Verify playback with DRM-enabled players

MAY:
- Implement key rotation policies
- Set up monitoring for key provider availability

## Common Issues

- symptoms: "Job fails with DRM key exchange error"
  diagnosis: "SPEKE key provider URL unreachable or returning errors."
  resolution: "Verify key provider URL. Check API Gateway configuration and Lambda function."

- symptoms: "DRM-protected content won't play"
  diagnosis: "DRM system mismatch with player or license server issue."
  resolution: "Verify DRM system matches player requirements. Check license server."

## Output Format

```yaml
root_cause: "drm — <specific_cause>"
evidence:
  - type: drm_config
    content: "<DRM settings>"
  - type: key_provider
    content: "<SPEKE endpoint status>"
severity: HIGH
mitigation:
  immediate: "Fix key provider or DRM configuration"
  long_term: "Implement DRM testing and key provider monitoring"
```


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

- **NEVER** delete or modify a production job queue while jobs are actively processing
- **NEVER** change IAM roles on job templates without verifying S3 input/output bucket access
- **NEVER** disable DRM or content protection settings on outputs destined for licensed content
- **NEVER** cancel reserved transcoding capacity without assessing active workload commitments
- **NEVER** modify output presets for compliance-regulated content without re-validation

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
