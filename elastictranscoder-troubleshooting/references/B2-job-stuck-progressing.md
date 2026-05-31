---
title: "B2 — Job Stuck in Progressing"
description: "Diagnose Elastic Transcoder jobs stuck in Progressing state"
status: active
severity: HIGH
triggers: ["job stuck", "job not completing", "progressing too long", "job hanging"]
owner: devops-agent
objective: "Resolve jobs that remain in Progressing state beyond expected duration"
context: "Jobs in Progressing state are actively processing. Large files or complex encoding take longer. Stuck jobs indicate input issues or service problems."
---

## Phase 1 — Triage

MUST:
- Read job details: `aws elastictranscoder read-job --id <job-id>`
- Check how long the job has been in Progressing state (compare with job creation time)
- Verify input file size and format: `aws s3api head-object --bucket <bucket> --key <input-key>`
- Check pipeline status: `aws elastictranscoder read-pipeline --id <pipeline-id>`

SHOULD:
- Compare with similar successful jobs for expected duration
- Check if other jobs on the same pipeline are also stuck: `aws elastictranscoder list-jobs-by-pipeline --pipeline-id <pipeline-id> --ascending`
- Verify input file is not being modified during transcoding

MAY:
- Check AWS Service Health Dashboard for Elastic Transcoder issues
- Review CloudWatch metrics for the pipeline

## Phase 2 — Remediate

MUST:
- Cancel the stuck job if it exceeds reasonable processing time: `aws elastictranscoder cancel-job --id <job-id>`
- Verify input file integrity before resubmitting
- Resubmit with a known-good preset to isolate preset vs input issues

SHOULD:
- Test with a smaller input file to verify pipeline is functioning
- Check if the input file codec is supported by the selected preset

MAY:
- Split large input files into smaller segments for parallel processing
- Consider using a different preset with lower output quality for faster processing

## Safety Ratings

```
safety_ratings:
  - "read-job: GREEN — read-only job inspection"
  - "cancel-job: YELLOW — terminates an active transcoding job"
  - "s3api head-object: GREEN — read-only file metadata"
  - "create-job: YELLOW — submits a new transcoding job"
```

## Escalation Conditions

- Multiple jobs stuck across different pipelines suggesting service-wide issue
- Job has been in Progressing state for more than 4 hours for a standard-length video
- Cancelling the job fails or the job remains in Progressing after cancellation
- Production media pipeline is blocked waiting for stuck jobs

## Data Sensitivity

- `aws elastictranscoder read-job`: LOW — returns job metadata and status
- `aws s3api head-object`: LOW — returns file metadata without content
- `aws elastictranscoder list-jobs-by-pipeline`: LOW — returns job listing

## Prohibited Actions

- NEVER cancel jobs without first checking if they are legitimately processing a large file
- NEVER delete the input file while a job is in Progressing state
- NEVER pause the pipeline to force-stop stuck jobs — this affects all jobs
- NEVER assume a stuck job will self-resolve without investigation

## Phase 3 — Rollback

1. If a job was cancelled, verify no partial output files remain: `aws s3 ls s3://<output-bucket>/<output-prefix>`
2. Clean up partial output files if present: `aws s3 rm s3://<output-bucket>/<partial-output-key>`
3. If the pipeline was paused during investigation, reactivate it: `aws elastictranscoder update-pipeline-status --id <pipeline-id> --status Active`
4. Resubmit the job with verified input and preset after root cause is identified
5. Document the stuck duration and root cause for capacity planning

## Common Issues

- symptoms: "Job in Progressing for over 2 hours for a 30-minute video"
  diagnosis: "Input file may be corrupted or use an unsupported codec variant."
  resolution: "Cancel the job. Re-encode the input file with a standard codec (H.264/AAC) and resubmit."

- symptoms: "All jobs on a pipeline are stuck in Progressing"
  diagnosis: "Pipeline may have hit concurrent job limit or service issue."
  resolution: "Check pipeline concurrent job count. If at limit, wait for jobs to complete or use another pipeline."

## Output Format

```yaml
root_cause: "job_stuck — <specific_cause>"
evidence:
  - type: job_duration
    content: "<time in Progressing state and expected duration>"
severity: HIGH
mitigation:
  immediate: "Cancel stuck job and resubmit"
  long_term: "Add job duration monitoring and alerting"
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
