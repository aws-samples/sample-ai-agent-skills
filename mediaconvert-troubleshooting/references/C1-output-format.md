---
title: "C1 — Output Format Issues"
description: "Diagnose output format and packaging configuration problems"
status: active
severity: MEDIUM
triggers:
  - "output format"
  - "HLS output"
  - "DASH output"
  - "CMAF"
  - "packaging"
owner: devops-agent
objective: "Resolve output format, packaging, and delivery configuration issues"
context: "MediaConvert supports multiple output group types: File Group, Apple HLS, DASH ISO, CMAF, and MS Smooth. Each has specific codec requirements and configuration options."
---

## Phase 1 — Triage

MUST:
- Check job output group configuration: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Verify output group type matches delivery requirements
- Check codec compatibility with the output group type
- Verify segment duration and naming settings for streaming outputs

SHOULD:
- Check manifest file generation settings
- Verify encryption settings if DRM is configured
- Review output modifier and naming template

MAY:
- Test output playback in target players
- Compare with working output configuration

## Phase 2 — Remediate

MUST:
- If HLS: ensure H.264 or H.265 video codec with AAC audio
- If DASH: verify codec and segment settings
- If CMAF: check fragment duration and codec compatibility
- If File Group: verify container and codec combination

SHOULD:
- Use standard segment durations (6s for HLS, 2-4s for DASH)
- Configure proper manifest naming
- Test with multiple players for compatibility

MAY:
- Implement ABR (Adaptive Bitrate) ladder with multiple renditions
- Use CMAF for unified HLS/DASH delivery

## Common Issues

- symptoms: "HLS manifest generated but segments missing"
  diagnosis: "Output path configuration or S3 write permission issue."
  resolution: "Check output S3 path and IAM role write permissions."

- symptoms: "DASH output not playing in browser"
  diagnosis: "Codec or segment configuration incompatible with player."
  resolution: "Use H.264 Baseline/Main profile for maximum compatibility."

## Output Format

```yaml
root_cause: "output_format — <specific_cause>"
evidence:
  - type: output_config
    content: "<output group settings>"
  - type: codec_settings
    content: "<codec configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix output format configuration"
  long_term: "Implement output templates and playback testing"
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
