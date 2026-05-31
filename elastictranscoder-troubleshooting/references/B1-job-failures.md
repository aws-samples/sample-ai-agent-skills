---
title: "B1 — Job Failures"
description: "Diagnose Elastic Transcoder job failures"
status: active
severity: HIGH
triggers: ["job failed", "transcoding error", "job status Error", "CreateJob failed"]
owner: devops-agent
objective: "Resolve transcoding job failures including input errors, codec issues, and permission problems"
context: "Jobs fail due to input file issues, unsupported codecs, permission errors, or preset mismatches."
---

## Phase 1 — Triage

MUST:
- Read job details: `aws elastictranscoder read-job --id <job-id>`
- Check job output status and error messages in the Output.StatusDetail field
- Verify input file exists: `aws s3 ls s3://<input-bucket>/<input-key>`
- Verify pipeline is active: `aws elastictranscoder read-pipeline --id <pipeline-id>`

SHOULD:
- Check preset compatibility with input file format: `aws elastictranscoder read-preset --id <preset-id>`
- Verify IAM role has s3:GetObject on input and s3:PutObject on output bucket
- List recent failed jobs: `aws elastictranscoder list-jobs-by-status --status Error --ascending`

MAY:
- Check input file metadata: `aws s3api head-object --bucket <bucket> --key <key>`
- Review CloudTrail for CreateJob events

## Phase 2 — Remediate

MUST:
- Fix input file issues (missing file, corrupted file, unsupported format)
- Correct preset selection to match input file format
- Fix IAM permissions if AccessDenied errors

SHOULD:
- Resubmit the job after fixing the root cause
- Verify output was written correctly after successful resubmission

MAY:
- Set up SNS notifications for Error status to catch future failures early

## Safety Ratings

```
safety_ratings:
  - "read-job: GREEN — read-only job inspection"
  - "list-jobs-by-status: GREEN — read-only job listing"
  - "s3 ls: GREEN — read-only bucket listing"
  - "create-job: YELLOW — submits a new transcoding job, consumes resources"
```

## Escalation Conditions

- Jobs fail consistently with internal service errors despite correct configuration
- Input files are valid but jobs fail with codec errors on supported formats
- Job failures affect production media pipeline with SLA requirements
- Multiple pipelines experiencing simultaneous job failures suggesting service issue

## Data Sensitivity

- `aws elastictranscoder read-job`: LOW — returns job metadata and status
- `aws s3 ls`: LOW — returns file listing only
- `aws s3api head-object`: LOW — returns file metadata without content
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER delete failed jobs without capturing error details first
- NEVER resubmit jobs in a loop without fixing the root cause
- NEVER modify the input file in S3 while other jobs may be reading it
- NEVER assume a job failure is transient without checking error details

## Phase 3 — Rollback

1. If a job produced partial or corrupted output, delete the output files: `aws s3 rm s3://<output-bucket>/<output-key>`
2. If multiple retry jobs were submitted, cancel pending ones: `aws elastictranscoder cancel-job --id <job-id>`
3. If pipeline configuration was changed to fix job issues, revert to original settings
4. Verify no duplicate outputs exist from retry attempts
5. Document the failure cause and resolution for future reference

## Common Issues

- symptoms: "Job fails with 'Could not find file' error"
  diagnosis: "Input file does not exist at the specified S3 key."
  resolution: "Verify the input key path. Ensure the file was uploaded before job submission."

- symptoms: "Job fails with AccessDenied"
  diagnosis: "IAM role lacks s3:GetObject on input bucket or s3:PutObject on output bucket."
  resolution: "Update IAM role policy to include required S3 permissions for both buckets."

## Output Format

```yaml
root_cause: "job_failure — <specific_cause>"
evidence:
  - type: job_error
    content: "<error details from StatusDetail>"
severity: HIGH
mitigation:
  immediate: "Fix root cause and resubmit job"
  long_term: "Add input validation before job submission"
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
