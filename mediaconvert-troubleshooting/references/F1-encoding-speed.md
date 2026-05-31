---
title: "F1 — Encoding Speed Issues"
description: "Diagnose slow encoding and performance optimization"
status: active
severity: MEDIUM
triggers:
  - "slow encoding"
  - "encoding speed"
  - "transcode time"
  - "performance"
owner: devops-agent
objective: "Optimize MediaConvert encoding speed and throughput"
context: "Encoding speed depends on input complexity, output settings, codec, resolution, rate control mode, and whether accelerated transcoding is used. Multi-pass encoding is slower but higher quality."
---

## Phase 1 — Triage

MUST:
- Check job timing: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>` (check timing in response)
- Review encoding settings (codec, resolution, passes, rate control)
- Check if accelerated transcoding is enabled
- Verify input file duration and complexity

SHOULD:
- Compare encoding time with similar jobs
- Check CloudWatch encoding time metrics
- Review if multi-pass encoding is necessary

MAY:
- Benchmark different encoding settings
- Check for queue wait time vs actual encoding time

## Phase 2 — Remediate

MUST:
- If eligible: enable accelerated transcoding (AccelerationMode: PREFERRED)
- If multi-pass not needed: switch to single-pass for faster encoding
- If resolution too high: match output resolution to delivery requirements

SHOULD:
- Use QVBR instead of multi-pass VBR for quality with speed
- Optimize encoding ladder (fewer renditions if possible)
- Use appropriate codec profile (Baseline is faster than High)

MAY:
- Split long content into segments for parallel processing
- Use job templates optimized for speed vs quality tradeoff

## Common Issues

- symptoms: "4K H.265 encoding takes hours"
  diagnosis: "H.265 encoding is computationally intensive, especially at 4K."
  resolution: "Enable accelerated transcoding. Use single-pass QVBR."

- symptoms: "Encoding time varies significantly for similar content"
  diagnosis: "Content complexity affects encoding time. Action scenes take longer."
  resolution: "This is expected. Use accelerated transcoding for consistent speed."

## Output Format

```yaml
root_cause: "encoding_speed — <specific_cause>"
evidence:
  - type: job_timing
    content: "<encoding duration>"
  - type: settings
    content: "<codec, resolution, passes>"
severity: MEDIUM
mitigation:
  immediate: "Enable accelerated transcoding or simplify settings"
  long_term: "Optimize encoding templates for speed/quality balance"
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
