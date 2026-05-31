---
title: "C2 — Quality Settings Issues"
description: "Diagnose video and audio quality configuration problems"
status: active
severity: MEDIUM
triggers:
  - "quality"
  - "bitrate"
  - "resolution"
  - "encoding quality"
  - "QVBR"
owner: devops-agent
objective: "Optimize encoding quality settings for MediaConvert jobs"
context: "Quality depends on resolution, bitrate, rate control mode (CBR, VBR, QVBR), codec profile/level, and encoding passes. QVBR is recommended for automatic quality optimization."
---

## Phase 1 — Triage

MUST:
- Check job encoding settings: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Review rate control mode (CBR, VBR, QVBR)
- Check resolution and bitrate relationship
- Verify codec profile and level settings

SHOULD:
- Compare output quality with source material
- Check if multi-pass encoding is enabled for quality-critical content
- Review audio bitrate and codec settings

MAY:
- Analyze output with quality metrics tools (VMAF, PSNR, SSIM)
- Compare with industry-standard encoding ladders

## Phase 2 — Remediate

MUST:
- If poor quality: increase bitrate or switch to QVBR with higher quality level
- If file too large: reduce bitrate or use QVBR for efficient encoding
- If resolution wrong: verify width/height settings and scaling behavior

SHOULD:
- Use QVBR rate control with quality level 7-9 for most content
- Match codec profile to target device compatibility
- Use 2-pass encoding for quality-critical content

MAY:
- Implement ABR encoding ladder following industry best practices
- Use per-title encoding for content-aware bitrate optimization

## Common Issues

- symptoms: "Output video looks blocky or pixelated"
  diagnosis: "Bitrate too low for the resolution and content complexity."
  resolution: "Increase bitrate or use QVBR with quality level 8-9."

- symptoms: "Output file size much larger than expected"
  diagnosis: "CBR with high bitrate or unnecessary resolution."
  resolution: "Switch to QVBR for efficient encoding. Match resolution to delivery requirements."

## Output Format

```yaml
root_cause: "quality — <specific_cause>"
evidence:
  - type: encoding_settings
    content: "<rate control, bitrate, resolution>"
  - type: quality_analysis
    content: "<quality comparison>"
severity: MEDIUM
mitigation:
  immediate: "Adjust quality settings and re-encode"
  long_term: "Implement QVBR-based encoding templates"
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
