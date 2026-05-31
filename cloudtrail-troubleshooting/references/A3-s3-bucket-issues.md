---
title: "A3 — S3 Bucket Issues"
description: "Diagnose S3 bucket configuration issues for CloudTrail"
status: active
severity: HIGH
triggers:
  - "bucket policy error"
  - "S3 access denied"
  - "bucket not found"
  - "log file location"
owner: devops-agent
objective: "Fix S3 bucket configuration for CloudTrail log delivery"
context: "CloudTrail requires specific S3 bucket policy permissions. The bucket must allow the CloudTrail service principal to PutObject and GetBucketAcl. Bucket encryption, lifecycle rules, and access settings can also affect log delivery and access."
---

## Phase 1 — Triage

MUST:
- Check bucket exists: `aws s3api head-bucket --bucket <bucket-name>`
- Check bucket policy: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Check bucket encryption: `aws s3api get-bucket-encryption --bucket <bucket-name>`
- Verify trail bucket configuration: `aws cloudtrail get-trail --name <trail-name> --query 'Trail.{Bucket:S3BucketName,Prefix:S3KeyPrefix}'`

SHOULD:
- Check bucket Block Public Access settings
- Verify bucket region matches trail region (for single-region trails)
- Check bucket lifecycle rules that might delete logs prematurely

MAY:
- Check bucket versioning configuration
- Review bucket access logging

## Phase 2 — Remediate

MUST:
- Set correct bucket policy with CloudTrail service principal permissions
- Ensure bucket exists and is accessible
- Fix encryption configuration if causing issues

SHOULD:
- Enable bucket versioning for log protection
- Set appropriate lifecycle rules for log retention
- Enable S3 Block Public Access

MAY:
- Enable S3 access logging on the CloudTrail bucket
- Configure cross-region replication for DR

## Common Issues

- symptoms: "InsufficientS3BucketPolicyException"
  diagnosis: "Bucket policy missing CloudTrail permissions."
  resolution: "Add s3:PutObject and s3:GetBucketAcl for cloudtrail.amazonaws.com."

- symptoms: "Logs delivered but cannot read them"
  diagnosis: "KMS encryption on bucket and reader lacks decrypt permission."
  resolution: "Grant kms:Decrypt to users who need to read CloudTrail logs."

- symptoms: "Old logs disappearing"
  diagnosis: "S3 lifecycle rule deleting objects."
  resolution: "Review and adjust lifecycle rules for appropriate retention."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set correct bucket policy | YELLOW | Bucket policy changes affect all access |
| Enable bucket versioning | GREEN | Additive protection, does not affect existing objects |
| Set lifecycle rules | YELLOW | May cause log deletion if misconfigured |
| Enable S3 Block Public Access | GREEN | Security enhancement |

## Escalation Conditions

- S3 bucket deleted or inaccessible — escalate immediately to security operations team
- Bucket policy changes in shared/centralized logging bucket — escalate to bucket owner
- Lifecycle rules deleting logs prematurely — escalate to compliance team
- Unauthorized bucket policy modifications detected — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws s3api get-bucket-policy` | HIGH | Exposes bucket access policy with account IDs |
| `aws s3api get-bucket-encryption` | MEDIUM | Exposes encryption configuration |
| `aws cloudtrail get-trail` | HIGH | Exposes trail configuration including bucket and prefix |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** set lifecycle rules that delete logs before compliance retention requirements are met
- **NEVER** disable bucket versioning on CloudTrail log buckets

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Updated bucket policy | Restore previous bucket policy from backup |
| Enabled bucket versioning | Cannot disable versioning (can only suspend); this is by design |
| Set lifecycle rules | Delete or modify lifecycle rules: `aws s3api delete-bucket-lifecycle` |
| Enabled Block Public Access | Disable specific Block Public Access settings if needed |

## Output Format

```yaml
root_cause: "s3_bucket — <specific_cause>"
evidence:
  - type: bucket_policy
    content: "<bucket policy>"
  - type: bucket_config
    content: "<encryption, lifecycle, versioning>"
severity: HIGH
mitigation:
  immediate: "Fix bucket policy and configuration"
  long_term: "Automate bucket configuration with IaC"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
