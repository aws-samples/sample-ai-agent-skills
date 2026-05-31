---
title: "D2 — Output Format Issues"
description: "Diagnose Elastic Transcoder output format and quality issues"
status: active
severity: MEDIUM
triggers: ["output format wrong", "output quality bad", "output file missing", "wrong resolution"]
owner: devops-agent
objective: "Resolve output format issues including missing files, quality problems, and incorrect encoding settings"
context: "Output format is determined by the preset. Output files are written to the pipeline output bucket with the configured key prefix."
---

## Phase 1 — Triage

MUST:
- Read job details for output status: `aws elastictranscoder read-job --id <job-id>`
- Verify output file exists: `aws s3 ls s3://<output-bucket>/<output-key-prefix>`
- Read the preset used for the job: `aws elastictranscoder read-preset --id <preset-id>`
- Check pipeline output bucket and key prefix: `aws elastictranscoder read-pipeline --id <pipeline-id>`

SHOULD:
- Verify output file size is reasonable for the content and settings
- Check if output key prefix in pipeline matches expected output location
- Compare output file metadata with preset settings

MAY:
- Download and inspect output file locally with ffprobe or mediainfo
- Compare output with a known-good transcode of the same input

## Phase 2 — Remediate

MUST:
- Fix preset settings if output format or quality is incorrect
- Correct output key prefix if files are in unexpected location
- Resubmit job with corrected preset after verification

SHOULD:
- Test preset changes with a short sample before full production run
- Verify output plays correctly in target player/device

MAY:
- Create multiple output presets for different quality tiers
- Set up automated quality checks on output files

## Safety Ratings

```
safety_ratings:
  - "read-job: GREEN — read-only job inspection"
  - "s3 ls: GREEN — read-only file listing"
  - "read-preset: GREEN — read-only preset inspection"
  - "create-job (resubmit): YELLOW — submits new transcoding job"
```

## Escalation Conditions

- Output files are consistently corrupted despite correct preset configuration
- Output quality is significantly below expected for the configured bitrate
- Output files are missing despite job showing Completed status
- Output format is correct but incompatible with target playback devices

## Data Sensitivity

- `aws elastictranscoder read-job`: LOW — returns job metadata
- `aws s3 ls`: LOW — returns file listing
- Output file content: HIGH — may contain proprietary media content

## Prohibited Actions

- NEVER delete output files without confirming they are not needed by downstream systems
- NEVER modify output files directly in S3 — resubmit the job instead
- NEVER assume output quality without verifying playback
- NEVER change pipeline output bucket while jobs are in progress

## Phase 3 — Rollback

1. If incorrect output files were produced, identify and remove them: `aws s3 rm s3://<output-bucket>/<incorrect-output-key>`
2. Restore previous output files from S3 versioning if available
3. Resubmit jobs with the correct preset to regenerate outputs
4. If pipeline output configuration was changed, revert to original settings
5. Verify downstream systems are consuming the correct output files

## Common Issues

- symptoms: "Output file exists but is in unexpected S3 location"
  diagnosis: "Pipeline output key prefix directs files to a different path than expected."
  resolution: "Check pipeline OutputBucket and OutputKeyPrefix. Update or adjust downstream file references."

- symptoms: "Output video quality is poor despite high bitrate preset"
  diagnosis: "Input file quality is lower than output preset settings — cannot upscale quality."
  resolution: "Match output preset quality to input quality. Transcoding cannot improve source quality."

## Output Format

```yaml
root_cause: "output_format — <specific_cause>"
evidence:
  - type: output_issue
    content: "<output file details and preset settings>"
severity: MEDIUM
mitigation:
  immediate: "Fix preset and resubmit"
  long_term: "Implement output quality validation"
```

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
