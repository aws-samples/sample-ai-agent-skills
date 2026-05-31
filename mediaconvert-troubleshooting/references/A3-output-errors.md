---
title: "A3 — Output Errors"
description: "Diagnose output file issues after job completion"
status: active
severity: MEDIUM
triggers:
  - "output error"
  - "wrong output"
  - "missing output"
  - "output quality"
owner: devops-agent
objective: "Resolve output file quality, format, and delivery issues"
context: "Output errors occur when the job completes but the output doesn't match expectations — wrong format, missing audio, poor quality, incorrect resolution, or files not appearing in the expected S3 location."
---

## Phase 1 — Triage

MUST:
- Check job output details: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Verify output files in S3: `aws s3 ls s3://<output-bucket>/<output-path>/`
- Check output group configuration in the job settings
- Verify output file can be played/opened

SHOULD:
- Compare output settings with expected results
- Check audio selector configuration for multi-track inputs
- Verify resolution and bitrate settings

MAY:
- Use MediaInfo or ffprobe to analyze output file properties
- Compare with a known-good output

## Phase 2 — Remediate

MUST:
- If missing output: check output S3 path and permissions
- If wrong format: verify output group type and codec settings
- If missing audio: configure audio selectors explicitly
- If poor quality: adjust bitrate, use QVBR, or increase quality level

SHOULD:
- Use job templates for consistent output configuration
- Test with a short clip before processing full content
- Validate output settings against delivery requirements

MAY:
- Implement automated output validation
- Use MediaConvert presets for standard output configurations

## Common Issues

- symptoms: "Output file has no audio"
  diagnosis: "Audio selectors not configured for multi-track input."
  resolution: "Add audio selectors specifying the desired audio tracks."

- symptoms: "Output quality is poor despite high resolution"
  diagnosis: "Bitrate too low for the resolution. Use QVBR for automatic optimization."
  resolution: "Increase bitrate or switch to QVBR rate control with quality level 7-9."

## Output Format

```yaml
root_cause: "output_error — <specific_cause>"
evidence:
  - type: job_output
    content: "<output configuration>"
  - type: output_files
    content: "<S3 output listing>"
severity: MEDIUM
mitigation:
  immediate: "Fix output configuration and resubmit"
  long_term: "Implement output validation and use job templates"
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
