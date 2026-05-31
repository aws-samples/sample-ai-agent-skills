---
title: "A1 — MediaConvert Job Failures"
description: "Diagnose MediaConvert job failures and error states"
status: active
severity: HIGH
triggers:
  - "job failed"
  - "job error"
  - "ERROR status"
  - "transcode failed"
owner: devops-agent
objective: "Identify and resolve the root cause of MediaConvert job failures"
context: "Job failures are indicated by ERROR status. The job details contain errorCode and errorMessage with the specific failure reason. Common causes include input file issues, codec problems, S3 access, and configuration errors."
---

## Phase 1 — Triage

MUST:
- Get endpoint: `aws mediaconvert describe-endpoints --query 'Endpoints[0].Url' --output text`
- Get job details: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Check errorCode and errorMessage in the job response
- List recent failed jobs: `aws mediaconvert list-jobs --status ERROR --max-results 10 --endpoint-url <endpoint>`

SHOULD:
- Check CloudWatch error metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConvert --metric-name JobsErrored --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Verify input file exists in S3: `aws s3 ls <input-s3-uri>`
- Check IAM role permissions

MAY:
- Review job template if using templates
- Check CloudTrail for API-level errors

## Phase 2 — Remediate

MUST:
- Error 1000 (input): verify input file exists and is accessible
- Error 1020 (codec): verify input codec is supported
- Error 1040 (S3): fix IAM role or bucket policy for S3 access
- Error 1060 (output): fix output configuration or S3 write permissions

SHOULD:
- Test with a known-good input file to isolate the issue
- Verify job settings match input file characteristics
- Check for corrupted input files

MAY:
- Use MediaConvert job templates for consistent configuration
- Implement job error notifications via CloudWatch Events

## Common Issues

- symptoms: "Job fails with error code 1040"
  diagnosis: "MediaConvert cannot access the S3 input or output bucket."
  resolution: "Verify IAM role has s3:GetObject for input and s3:PutObject for output."

- symptoms: "Job fails with error code 1020"
  diagnosis: "Input file codec is not supported by MediaConvert."
  resolution: "Check supported codecs. Re-encode the input to a supported format."

## Output Format

```yaml
root_cause: "job_failure — <error_code> — <specific_cause>"
evidence:
  - type: job_error
    content: "<errorCode and errorMessage>"
  - type: job_settings
    content: "<relevant job configuration>"
severity: HIGH
mitigation:
  immediate: "Fix the identified error and resubmit the job"
  long_term: "Implement input validation and error notification"
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
