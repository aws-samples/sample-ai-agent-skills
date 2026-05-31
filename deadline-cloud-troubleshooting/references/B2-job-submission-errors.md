---
title: "B2 — Job Submission Errors"
description: "Diagnose job submission failures and validation errors"
status: active
severity: HIGH
triggers: ["job submission failed", "cannot submit job", "CreateJob error", "job validation error"]
owner: devops-agent
objective: "Resolve job submission failures"
context: "Job submission involves validating the job template, uploading attachments to S3, and creating the job in the queue. Failures can occur at validation, upload, or creation stages."
---

## Phase 1 — Triage

MUST:
- Check recent job submissions: `aws deadline list-jobs --farm-id <farm-id> --queue-id <queue-id> --query 'jobs[*].{JobId:jobId,Name:name,LifecycleStatus:lifecycleStatus,LifecycleStatusMessage:lifecycleStatusMessage}'`
- Verify queue exists and is active
- Check IAM permissions for deadline:CreateJob

SHOULD:
- Validate job template syntax (OpenJD format)
- Verify job attachment S3 bucket is accessible
- Check job parameters match template requirements

MAY:
- Review CloudTrail for CreateJob errors
- Test with a minimal job template to isolate the issue

## Phase 2 — Remediate

MUST:
- Fix job template syntax errors (OpenJD specification compliance)
- Ensure submitter has deadline:CreateJob permission on the queue
- Verify S3 bucket for job attachments is accessible

SHOULD:
- Validate job parameters before submission
- Ensure job attachment files exist and are readable

MAY:
- Use Deadline Cloud CLI for submission with better error messages
- Configure default job parameters in queue settings

## Safety Ratings

```
safety_ratings:
  - "list-jobs: GREEN — read-only job enumeration"
  - "create-job: YELLOW — submits job for execution with compute cost"
  - "cancel-job: YELLOW — stops job execution, may waste completed work"
  - "CloudTrail lookup: GREEN — read-only audit log"
```

## Escalation Conditions

- Job submission fails with internal service errors after fixing all parameters
- S3 bucket access requires cross-account permissions
- Job template requires features not supported in current Deadline Cloud version
- Bulk job submission failures affecting production render pipeline

## Data Sensitivity

- Job templates: MEDIUM — reveal render pipeline configuration
- Job parameters: MEDIUM — may contain file paths and project names
- Job attachments: HIGH — may contain proprietary creative assets
- S3 bucket contents: HIGH — render assets and output files

## Prohibited Actions

- NEVER submit jobs with overly broad IAM permissions
- NEVER expose job attachment S3 bucket publicly
- NEVER submit untested job templates to production queues
- NEVER cancel running jobs without confirming with the submitting artist

## Phase 3 — Rollback

1. If job was submitted with wrong parameters, cancel it: `aws deadline update-job --farm-id <id> --queue-id <id> --job-id <id> --target-task-run-status CANCELED`
2. If job attachments were uploaded to wrong bucket, clean up S3 objects
3. If job template was modified, revert to previous version
4. Verify queue is accepting new submissions after fix

## Common Issues

- symptoms: "CreateJob returns ValidationException"
  diagnosis: "Job template has syntax errors or missing required fields."
  resolution: "Validate job template against OpenJD specification. Check required fields: name, steps, script."

- symptoms: "Job created but immediately fails with UPLOAD_FAILED"
  diagnosis: "Job attachments cannot be uploaded to S3."
  resolution: "Verify S3 bucket exists and queue IAM role has s3:PutObject permission. Check file paths."

## Output Format

```yaml
root_cause: "job_submission — <specific_cause>"
evidence:
  - type: submission_error
    content: "<error details and job template>"
severity: HIGH
mitigation:
  immediate: "Fix submission parameters or template"
  long_term: "Implement job template validation in CI/CD"
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
