---
title: "B2 — Codec Support Issues"
description: "Diagnose codec and format compatibility problems"
status: active
severity: MEDIUM
triggers:
  - "codec not supported"
  - "unsupported format"
  - "error 1020"
  - "codec error"
owner: devops-agent
objective: "Resolve codec and format compatibility issues for MediaConvert jobs"
context: "MediaConvert supports specific input and output codecs. Error code 1020 indicates unsupported codec. Some codecs are input-only, some output-only. Container/codec combinations must be valid."
---

## Phase 1 — Triage

MUST:
- Check job error for codec details: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Identify input file codec (use MediaInfo or ffprobe on the source file)
- Verify the codec is in MediaConvert's supported list
- Check the output codec/container combination is valid

SHOULD:
- Verify input container format is supported
- Check for codec profile/level compatibility
- Review output group requirements for codec support

MAY:
- Check if the codec requires a specific MediaConvert tier
- Review AWS documentation for latest codec support

## Phase 2 — Remediate

MUST:
- If input codec unsupported: re-encode source to a supported codec (H.264, H.265)
- If output codec/container mismatch: use a compatible combination
- If codec profile unsupported: adjust profile/level settings

SHOULD:
- Use widely supported codecs (H.264/AAC) for maximum compatibility
- Test codec combinations with short clips before full processing
- Document supported codec matrix for the team

MAY:
- Use ffmpeg to pre-process unsupported inputs
- Implement codec validation in the upload pipeline

## Common Issues

- symptoms: "Error 1020: Unsupported input codec"
  diagnosis: "Input file uses a codec not supported by MediaConvert."
  resolution: "Re-encode input to H.264/AAC using ffmpeg before submitting to MediaConvert."

- symptoms: "HLS output fails with VP9 codec"
  diagnosis: "HLS requires H.264 or H.265 video codec. VP9 is not supported for HLS."
  resolution: "Use H.264 or H.265 for HLS output groups."

## Output Format

```yaml
root_cause: "codec — <specific_cause>"
evidence:
  - type: input_codec
    content: "<detected codec>"
  - type: error_details
    content: "<error code and message>"
severity: MEDIUM
mitigation:
  immediate: "Re-encode input or fix codec configuration"
  long_term: "Implement codec validation in upload pipeline"
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
