---
title: "E1 — Batch Processing Issues"
description: "Diagnose async batch job failures"
status: active
severity: MEDIUM
triggers:
  - "batch job failed"
  - "Start*Job error"
  - "async job not completing"
owner: devops-agent
objective: "Identify and resolve batch processing issues"
context: "Async batch jobs process documents from S3. Require IAM data access role. Input format varies by job type. Jobs can take minutes to hours depending on volume."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws comprehend describe-entities-detection-job --job-id <job-id>`
- Verify S3 input exists and is accessible
- Check IAM data access role has S3 read/write permissions
- Review job error message if failed

SHOULD:
- Verify input format (ONE_DOC_PER_LINE or ONE_DOC_PER_FILE)
- Check S3 output location is writable
- Verify language code parameter

MAY:
- Check for concurrent job limits
- Monitor job processing time

## Phase 2 — Remediate

MUST:
- Fix S3 permissions on data access role
- Use correct input format
- Specify correct language code

SHOULD:
- Test with small input before large batch
- Monitor job progress

MAY:
- Implement batch job orchestration pipeline

## Common Issues

- symptoms: "Job fails with access denied"
  diagnosis: "Data access role lacks S3 permissions."
  resolution: "Add s3:GetObject and s3:PutObject to data access role."

- symptoms: "Job completes but output is empty"
  diagnosis: "Input format mismatch or no content detected."
  resolution: "Check input format matches InputFormat parameter."

## Output Format

```yaml
root_cause: "batch_processing — <specific_cause>"
evidence:
  - type: job_status
    content: "<job details>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions or input format"
  long_term: "Implement batch job monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Batch jobs process documents from S3 that may contain sensitive content |
| Blast Radius | MEDIUM — Failed jobs waste compute; incorrect output affects downstream systems |
| Rollback Complexity | LOW — Resubmit jobs after fixing configuration |
| Compliance Scope | MEDIUM — Batch outputs must be secured per data classification |

## Escalation Conditions

- Escalate to security if the data access role has overly broad S3 permissions
- Escalate to management if repeated job failures consume significant budget
- Escalate to data governance if batch output is written to unsecured S3 locations

## Data Sensitivity

- **S3 input data** may contain sensitive documents — verify bucket encryption and access policies
- **S3 output data** contains NLP analysis results — apply same sensitivity as input
- **IAM data access role** has S3 read/write — scope to specific buckets and prefixes
- **Job metadata** in CloudTrail does not contain document content

## PROHIBITED Actions

- Do NOT grant the data access role s3:* permissions — use least-privilege
- Do NOT write batch output to publicly accessible S3 buckets
- Do NOT run batch jobs on production data without testing on sample data first
- Do NOT leave failed job outputs in S3 without cleanup

## Phase 3 — Rollback

- Stop running jobs if incorrect: `aws comprehend stop-entities-detection-job --job-id <job-id>`
- Delete incorrect output from S3
- Fix job configuration (input format, language code, permissions)
- Resubmit the job with corrected parameters

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
