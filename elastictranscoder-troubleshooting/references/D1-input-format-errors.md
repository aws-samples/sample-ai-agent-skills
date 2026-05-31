---
title: "D1 — Input Format Errors"
description: "Diagnose Elastic Transcoder input file format errors"
status: active
severity: HIGH
triggers: ["input format error", "unsupported format", "cannot read input", "input file invalid"]
owner: devops-agent
objective: "Resolve input file format issues including unsupported codecs, corrupted files, and container mismatches"
context: "Elastic Transcoder supports specific input formats. Input files must be in S3 and accessible by the pipeline IAM role."
---

## Phase 1 — Triage

MUST:
- Read job error details: `aws elastictranscoder read-job --id <job-id>` — check Output.StatusDetail
- Verify input file exists and is accessible: `aws s3 ls s3://<input-bucket>/<input-key>`
- Check input file size: `aws s3api head-object --bucket <bucket> --key <input-key>`
- Verify the file is not zero bytes or truncated

SHOULD:
- Check input file extension matches actual format
- Verify the input codec is supported (H.264, MPEG-2, VP8, VP9, AAC, MP3, FLAC, WAV, etc.)
- Test with a known-good input file to isolate file vs configuration issues

MAY:
- Download and inspect the input file locally with ffprobe or mediainfo
- Check if the file was uploaded completely (compare local and S3 file sizes)

## Phase 2 — Remediate

MUST:
- Re-encode unsupported input files to a supported format (H.264/AAC in MP4 recommended)
- Re-upload corrupted or truncated files
- Ensure input file upload is complete before submitting jobs

SHOULD:
- Add input validation to the upload pipeline to reject unsupported formats early
- Use multipart upload for large files to prevent truncation

MAY:
- Set up S3 event notifications to trigger transcoding only after upload completion
- Create a pre-processing Lambda to validate input files before job submission

## Safety Ratings

```
safety_ratings:
  - "read-job: GREEN — read-only job inspection"
  - "s3 ls: GREEN — read-only file listing"
  - "s3api head-object: GREEN — read-only file metadata"
  - "s3 cp (re-upload): YELLOW — writes data to S3"
```

## Escalation Conditions

- Input format is listed as supported but consistently fails with format errors
- Large files fail while small files of the same format succeed
- Input files from a specific source consistently fail suggesting encoding issues
- Format errors occur intermittently for the same file

## Data Sensitivity

- `aws elastictranscoder read-job`: LOW — returns job metadata
- `aws s3 ls`: LOW — returns file listing
- `aws s3api head-object`: LOW — returns file metadata
- Input file content: HIGH — may contain proprietary or sensitive media content

## Prohibited Actions

- NEVER delete the original input file before confirming a successful transcode
- NEVER modify the input file in S3 while jobs are processing it
- NEVER assume file format based on extension alone — verify actual codec
- NEVER re-encode input files without preserving the original

## Phase 3 — Rollback

1. If a re-encoded input file was uploaded, preserve the original: verify original file is still in S3
2. If jobs were submitted with the wrong input key, cancel them: `aws elastictranscoder cancel-job --id <job-id>`
3. Clean up any test or re-encoded files uploaded during troubleshooting
4. Restore the original workflow if input validation changes were made
5. Document the supported format requirements for the content pipeline

## Common Issues

- symptoms: "Job fails with 'Could not interpret the input file as a supported format'"
  diagnosis: "Input file uses an unsupported codec or is corrupted."
  resolution: "Re-encode the input file using H.264 video and AAC audio in an MP4 container."

- symptoms: "Job fails with zero-byte input file"
  diagnosis: "File upload was incomplete or file is empty."
  resolution: "Re-upload the file. Use multipart upload for large files. Verify file size after upload."

## Output Format

```yaml
root_cause: "input_format — <specific_cause>"
evidence:
  - type: format_error
    content: "<error details and file metadata>"
severity: HIGH
mitigation:
  immediate: "Fix input file and resubmit"
  long_term: "Add input format validation to upload pipeline"
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
