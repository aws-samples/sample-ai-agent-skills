---
title: "F1 — Watermark and Thumbnail Issues"
description: "Diagnose Elastic Transcoder watermark and thumbnail generation issues"
status: active
severity: MEDIUM
triggers: ["watermark missing", "thumbnail not generated", "watermark wrong position", "thumbnail quality"]
owner: devops-agent
objective: "Resolve watermark rendering and thumbnail generation issues"
context: "Watermarks require PNG with alpha channel. Thumbnails are configured per-output with interval, format, and size settings."
---

## Phase 1 — Triage

MUST:
- Read job details for output and thumbnail status: `aws elastictranscoder read-job --id <job-id>`
- Read preset for watermark and thumbnail configuration: `aws elastictranscoder read-preset --id <preset-id>`
- Verify watermark file exists in S3: `aws s3 ls s3://<input-bucket>/<watermark-key>`
- Check thumbnail output location: `aws s3 ls s3://<thumbnail-bucket>/<thumbnail-prefix>`

SHOULD:
- Verify watermark file is PNG format with alpha channel
- Check watermark file size is reasonable (not corrupted or zero bytes)
- Verify thumbnail bucket is configured in the pipeline: `aws elastictranscoder read-pipeline --id <pipeline-id>`

MAY:
- Download and inspect watermark file locally for format verification
- Compare thumbnail output with preset thumbnail settings

## Phase 2 — Remediate

MUST:
- Convert watermark images to PNG with alpha channel transparency
- Configure thumbnail settings in the output section of the job (not job-level)
- Ensure thumbnail bucket is specified in pipeline configuration

SHOULD:
- Set appropriate watermark sizing: MaxWidth, MaxHeight, SizingPolicy, HorizontalAlign, VerticalAlign
- Configure thumbnail interval, format (png or jpg), and max size
- Test watermark positioning with a short sample video

MAY:
- Create multiple watermark configurations for different output resolutions
- Set up automated thumbnail quality checks

## Safety Ratings

```
safety_ratings:
  - "read-job: GREEN — read-only job inspection"
  - "read-preset: GREEN — read-only preset inspection"
  - "s3 ls: GREEN — read-only file listing"
  - "create-job (test): YELLOW — submits test transcoding job"
```

## Escalation Conditions

- Watermarks render incorrectly despite correct PNG format and configuration
- Thumbnails are generated but are blank or corrupted
- Watermark positioning is inconsistent across different input resolutions
- Thumbnail generation works for some outputs but not others in the same job

## Data Sensitivity

- `aws elastictranscoder read-job`: LOW — returns job metadata
- `aws elastictranscoder read-preset`: LOW — returns preset configuration
- Watermark files: MEDIUM — may contain branded or proprietary imagery
- Thumbnail files: MEDIUM — may contain frames from proprietary video content

## Prohibited Actions

- NEVER delete watermark source files without confirming they are backed up
- NEVER modify watermark files in S3 while jobs are processing
- NEVER assume watermark format based on file extension — verify actual format
- NEVER disable thumbnail generation on production presets without approval

## Phase 3 — Rollback

1. If watermark files were replaced, restore originals from backup
2. If preset thumbnail settings were changed, revert to original configuration
3. Clean up test output files generated during troubleshooting: `aws s3 rm s3://<output-bucket>/<test-prefix> --recursive`
4. If pipeline thumbnail bucket was changed, restore original bucket reference
5. Verify production jobs produce correct watermarks and thumbnails after changes

## Common Issues

- symptoms: "Watermark not visible in output video"
  diagnosis: "Watermark file is JPEG instead of PNG, or PNG lacks alpha channel."
  resolution: "Convert watermark to PNG with alpha channel transparency. Re-upload and resubmit job."

- symptoms: "Thumbnails not generated despite preset configuration"
  diagnosis: "Thumbnail settings are in the preset but thumbnail bucket is not configured in the pipeline."
  resolution: "Update pipeline to include ThumbnailBucket. Thumbnails are written to this bucket, not the output bucket."

## Output Format

```yaml
root_cause: "watermark_thumbnail — <specific_cause>"
evidence:
  - type: media_feature_issue
    content: "<watermark/thumbnail configuration and file details>"
severity: MEDIUM
mitigation:
  immediate: "Fix watermark/thumbnail configuration"
  long_term: "Validate media assets before job submission"
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
