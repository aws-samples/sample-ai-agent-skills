---
title: "A1 — Discovery Job Failures"
description: "Diagnose why Macie discovery jobs are failing"
status: active
severity: HIGH
triggers:
  - "job failed"
  - "discovery job error"
  - "classification job stuck"
  - "job not running"
owner: devops-agent
objective: "Identify why Macie discovery jobs are failing and restore scanning"
context: "Discovery jobs can fail due to S3 access permissions, bucket policy restrictions, KMS key access, service-linked role issues, or bucket configuration problems. Check job status and error details for specific failure reasons."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws macie2 describe-classification-job --job-id <job-id>`
- List recent jobs: `aws macie2 list-classification-jobs --sort-criteria '{"attributeName":"createdAt","orderBy":"DESC"}' --max-results 10`
- Check Macie service-linked role: `aws iam get-role --role-name AWSServiceRoleForAmazonMacie`
- Check S3 bucket access: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Check if bucket is encrypted with KMS and Macie has key access
- Verify bucket is in the same region as Macie
- Check CloudTrail for job-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=macie2.amazonaws.com --max-results 20`

MAY:
- Check if bucket has S3 Object Lock or Glacier objects (may affect scanning)
- Review job scope configuration for incorrect bucket/prefix settings

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions to allow Macie service-linked role access
- Grant KMS key access to Macie service-linked role if bucket uses CMK
- Verify service-linked role exists and has correct permissions

SHOULD:
- Re-run failed job after fixing permissions
- Test with a small bucket first to verify access
- Check job scope to ensure correct buckets are targeted

MAY:
- Create a new job if the existing one cannot be recovered
- Set up job event notifications for failure alerting

## Common Issues

- symptoms: "Job status COMPLETE but no findings"
  diagnosis: "No sensitive data found, or job scope too narrow."
  resolution: "Verify job scope includes correct buckets/prefixes. Check managed identifiers are enabled."

- symptoms: "Job fails with access denied"
  diagnosis: "Macie service-linked role cannot access the S3 bucket or KMS key."
  resolution: "Update bucket policy and KMS key policy to allow Macie access."

- symptoms: "Job stuck in RUNNING for too long"
  diagnosis: "Large bucket with many objects taking time to process."
  resolution: "Check job progress. Large buckets can take hours. Consider sampling."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix S3 bucket permissions | YELLOW | Bucket policy changes affect all access |
| Grant KMS key access to Macie | YELLOW | Key policy changes affect all key usage |
| Re-run failed job | GREEN | Creates new scan, non-destructive |
| Create new job | GREEN | New resource creation |

## Escalation Conditions

- Discovery jobs failing on compliance-critical buckets — escalate to compliance team
- S3 bucket policy changes required — escalate to bucket owner
- KMS key policy changes required — escalate to key administrator
- Persistent job failures across multiple buckets — escalate to AWS Support

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 describe-classification-job` | HIGH | Exposes job configuration and target buckets |
| `aws macie2 list-classification-jobs` | HIGH | Exposes all job configurations |
| `aws s3api get-bucket-policy` | HIGH | Exposes bucket access policy |
| `aws cloudtrail lookup-events` (Macie) | HIGH | Exposes API call history |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** modify S3 bucket policies to make buckets publicly accessible
- **NEVER** grant overly broad KMS key access

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed S3 bucket policy | Restore previous bucket policy |
| Granted KMS key access | Remove Macie from KMS key policy |
| Created new discovery job | Cancel or delete the job: `aws macie2 update-classification-job --job-id <id> --job-status CANCELLED` |

## Output Format

```yaml
root_cause: "discovery_job — <specific_cause>"
evidence:
  - type: job_status
    content: "<job status and error details>"
  - type: permissions
    content: "<S3 and KMS access status>"
severity: HIGH
mitigation:
  immediate: "Fix permissions and re-run the job"
  long_term: "Automate permission validation before job creation"
```

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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
