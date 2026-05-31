---
title: "E1 — Batch Inference Issues"
description: "Diagnose batch inference job failures"
status: active
severity: MEDIUM
triggers: ["batch inference failed", "CreateBatchInferenceJob error", "batch job error"]
owner: devops-agent
objective: "Identify and resolve batch inference failures"
context: "Batch inference generates recommendations for many users from S3 input. Requires JSON lines format. Output written to S3. Requires active solution version."
---

## Phase 1 — Triage

MUST:
- Check job: `aws personalize describe-batch-inference-job --batch-inference-job-arn <arn>`
- Verify S3 input format (JSON lines: {"userId":"<id>"} per line)
- Check S3 permissions for IAM role
- Verify solution version is ACTIVE

SHOULD:
- Check input file encoding (UTF-8)
- Verify output S3 location is writable
- Review job failure reason

MAY:
- Test with small input file first
- Check for concurrent job limits

## Phase 2 — Remediate

MUST:
- Use correct JSON lines format in S3 input
- Grant S3 read/write to IAM role
- Use ACTIVE solution version

SHOULD:
- Validate input file before submission
- Monitor job progress

MAY:
- Implement batch inference pipeline

## Common Issues

- symptoms: "Batch job fails with input error"
  diagnosis: "Input format not JSON lines or invalid user IDs."
  resolution: "Use {\"userId\":\"<id>\"} format, one per line."

- symptoms: "Batch job fails with access denied"
  diagnosis: "IAM role lacks S3 permissions."
  resolution: "Add s3:GetObject and s3:PutObject to role."

## Output Format

```yaml
root_cause: "batch_inference — <specific_cause>"
evidence:
  - type: job_status
    content: "<batch job details>"
severity: MEDIUM
mitigation:
  immediate: "Fix input format or permissions"
  long_term: "Implement batch inference pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — batch inference processes user data at scale
- Service Disruption Risk: LOW — batch failures do not affect real-time recommendations
- Compliance Impact: HIGH — batch processing must comply with data handling policies

## Escalation Conditions
- Batch inference job fails with data format errors
- Batch job produces incorrect recommendations
- Batch output contains PII that should be anonymized
- Batch job exceeds expected duration or cost
- S3 output permissions prevent result delivery

## Data Sensitivity
- Input user data — RESTRICTED
- Batch recommendation output — CONFIDENTIAL
- S3 input/output paths — INTERNAL
- Job configurations — INTERNAL
- Never expose batch input or output data

## Prohibited Actions
- Do NOT run batch inference on unvalidated input data
- Do NOT store batch output in publicly accessible S3 buckets
- Do NOT ignore batch job failures
- Do NOT process data without proper consent
- Do NOT retain batch output beyond approved retention periods
- Do NOT share batch results with unauthorized parties

## Phase 3 — Rollback
- Delete incorrect batch output
- Re-run batch job with corrected input
- Restore S3 permissions if modified
- Verify batch output quality after re-run
- Restore previous batch configurations
- Document batch job issues and data handling procedures

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
