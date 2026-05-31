---
title: "C1 — S3 Export Failures"
description: "Diagnose S3 data export job failures in Data Exchange"
status: active
severity: HIGH
triggers: ["export job failed", "S3 export error", "job stuck", "cannot export data", "EXPORT_ASSETS_TO_S3 error"]
owner: devops-agent
objective: "Resolve S3 export job failures including permissions, bucket configuration, and job errors"
context: "S3 export jobs copy data from provider's data set to subscriber's S3 bucket. Jobs are asynchronous and require correct S3 permissions on the destination bucket."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws dataexchange get-job --job-id <job-id>`
- List failed jobs: `aws dataexchange list-jobs --query 'Jobs[?State==`ERROR`]'`
- Verify destination S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <bucket-name>`

SHOULD:
- Check IAM permissions for s3:PutObject on destination bucket
- Verify the data set has finalized revisions with assets
- Review job error details in the job response

MAY:
- Check S3 bucket policy for cross-account restrictions
- Verify S3 bucket encryption configuration compatibility

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions (IAM policy and bucket policy)
- Ensure destination bucket exists in the correct region
- Retry failed export jobs after fixing the root cause

SHOULD:
- Verify S3 bucket does not have restrictive bucket policies blocking Data Exchange
- Check that S3 bucket versioning and encryption settings are compatible

MAY:
- Configure S3 lifecycle rules for exported data management
- Set up CloudWatch alarms for export job failures

## Safety Ratings

```
safety_ratings:
  - "dataexchange get-job: GREEN — read-only job status check"
  - "dataexchange list-jobs: GREEN — read-only job listing"
  - "s3api head-bucket: GREEN — read-only bucket check"
  - "dataexchange create-job (export): YELLOW — writes data to S3 bucket, incurs transfer costs"
```

## Escalation Conditions

- Export jobs fail with internal service errors after permission fixes
- Large data set exports consistently timeout
- S3 bucket policy restrictions require account administrator changes
- Export job creates partial data and fails mid-transfer

## Data Sensitivity

- `aws dataexchange get-job`: MEDIUM — reveals S3 bucket, key paths, and data set details
- `aws dataexchange list-jobs`: LOW — returns job metadata
- `aws s3api head-bucket`: LOW — returns bucket existence status
- `exported S3 data`: HIGH — contains the actual subscribed data

## Prohibited Actions

- NEVER export data to publicly accessible S3 buckets
- NEVER export data to S3 buckets without appropriate encryption
- NEVER delete partially exported data without verifying job completion status
- NEVER share export job details containing S3 paths in public channels

## Phase 3 — Rollback

1. If data was exported to wrong bucket, delete exported objects: `aws s3 rm s3://<bucket>/<prefix> --recursive`
2. If export created partial data, delete and re-export after fixing the issue
3. Verify S3 bucket permissions are restored to intended state
4. Confirm no unauthorized access to exported data occurred
5. Re-run export job to correct destination

## Common Issues

- symptoms: "Export job fails — Access Denied writing to S3 bucket"
  diagnosis: "IAM role lacks s3:PutObject permission on the destination bucket."
  resolution: "Add s3:PutObject permission for the Data Exchange service principal on the bucket."

## Output Format

```yaml
root_cause: "s3_export — <specific_cause>"
evidence:
  - type: export_error
    content: "<error details from get-job>"
severity: HIGH
mitigation:
  immediate: "Fix export permissions and retry"
  long_term: "Automate export with proper IAM and monitoring"
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
