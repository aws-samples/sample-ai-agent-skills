---
title: "F2 — S3 Output Configuration"
description: "Diagnose S3 output configuration issues for Contact Lens and Connect"
status: active
severity: HIGH
triggers: ["S3 output error", "recordings not in S3", "analytics not in S3", "bucket permission error"]
owner: devops-agent
objective: "Resolve S3 bucket configuration issues for Contact Lens output, recordings, and analytics"
context: "Amazon Connect stores call recordings, Contact Lens analytics, screen recordings, and exported data in S3. The S3 bucket must grant write permissions to the Connect service. Bucket configuration affects all data storage."
---

## Phase 1 — Triage

MUST:
- Check the S3 bucket associated with the Connect instance
- Verify bucket exists and is accessible: `aws s3api head-bucket --bucket <bucket-name>`
- Check bucket policy for Connect service permissions: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Verify the bucket is in the same region as the Connect instance
- Check for S3 lifecycle rules that might delete data prematurely: `aws s3api get-bucket-lifecycle-configuration --bucket <bucket-name>`
- Verify KMS key permissions if server-side encryption is enabled

MAY:
- Check S3 bucket metrics for write errors: `aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name 4xxErrors --dimensions Name=BucketName,Value=<bucket-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Review CloudTrail for S3 access denied events

## Phase 2 — Remediate

MUST:
- Add Connect service principal to bucket policy: ensure connect.amazonaws.com has s3:PutObject and s3:GetBucketAcl permissions
- Verify the bucket is in the same AWS region as the Connect instance
- If using KMS encryption, grant the Connect service access to the KMS key

SHOULD:
- Configure appropriate lifecycle rules for data retention compliance
- Enable S3 versioning for data protection
- Set up S3 event notifications for monitoring

MAY:
- Enable S3 access logging for audit compliance
- Configure cross-region replication for disaster recovery
- Implement S3 Intelligent-Tiering for cost optimization

## Safety Ratings

```
safety_ratings:
  - "s3api head-bucket: GREEN — read-only bucket existence check"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "s3api put-bucket-policy: RED — modifies bucket access control"
  - "s3api get-bucket-lifecycle-configuration: GREEN — read-only lifecycle rules"
  - "KMS key policy modification: RED — affects encryption access control"
```

## Escalation Conditions

- Bucket policy changes affect other services using the same bucket
- KMS key policy changes require key administrator access
- Data retention requirements conflict with storage cost constraints
- Cross-account S3 access is required for centralized data storage

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access control
- S3 bucket contents: CRITICAL — contains call recordings, transcripts, and customer data
- KMS key policy: HIGH — controls encryption access
- S3 lifecycle rules: MEDIUM — reveals data retention policies

## Prohibited Actions

- NEVER remove existing bucket policy entries without understanding their purpose
- NEVER disable encryption on buckets containing call recordings or transcripts
- NEVER set bucket ACL to public for any Connect data bucket
- NEVER delete S3 objects without verifying retention requirements

## Phase 3 — Rollback

1. Revert bucket policy to previous version if changes caused access issues
2. Restore KMS key policy if encryption changes broke data access
3. Revert lifecycle rule changes if they caused premature data deletion
4. Verify Connect can write to the bucket after any changes
5. Document the S3 configuration change and rollback reason

## Common Issues

- symptoms: "Call recordings and analytics not appearing in S3"
  diagnosis: "S3 bucket policy does not grant connect.amazonaws.com write permissions."
  resolution: "Add connect.amazonaws.com as an allowed principal with s3:PutObject permission."

## Output Format

```yaml
root_cause: "s3_output_configuration — <specific_cause>"
evidence:
  - type: s3_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 bucket permissions"
  long_term: "Implement S3 monitoring and automated policy validation"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
