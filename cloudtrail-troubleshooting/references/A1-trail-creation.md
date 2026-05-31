---
title: "A1 — Trail Creation"
description: "Diagnose trail creation failures"
status: active
severity: HIGH
triggers:
  - "trail creation failed"
  - "cannot create trail"
  - "CreateTrail error"
owner: devops-agent
objective: "Identify why CloudTrail trail creation is failing and resolve the issue"
context: "Trail creation can fail due to S3 bucket permissions, KMS key access, IAM permissions, trail name conflicts, or region-specific issues. The S3 bucket must exist and have the correct policy before trail creation."
---

## Phase 1 — Triage

MUST:
- Check existing trails: `aws cloudtrail describe-trails`
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <bucket-name>`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names cloudtrail:CreateTrail`

SHOULD:
- Check if trail name already exists
- Verify KMS key exists and has correct policy (if using encryption)
- Check CloudWatch Logs log group exists (if configuring CW integration)
- Review CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTrail --max-results 5`

MAY:
- Check service quotas for maximum trails per region
- Verify SNS topic exists (if configuring notifications)

## Phase 2 — Remediate

MUST:
- Create S3 bucket with correct policy allowing CloudTrail service principal
- Fix IAM permissions for trail creation
- Use unique trail name

SHOULD:
- Create trail: `aws cloudtrail create-trail --name <name> --s3-bucket-name <bucket> --is-multi-region-trail --enable-log-file-validation`
- Start logging: `aws cloudtrail start-logging --name <trail-name>`

MAY:
- Add KMS encryption: `--kms-key-id <key-arn>`
- Add CloudWatch Logs: `--cloud-watch-logs-log-group-arn <arn> --cloud-watch-logs-role-arn <role-arn>`

## Common Issues

- symptoms: "InsufficientS3BucketPolicyException"
  diagnosis: "S3 bucket policy doesn't allow CloudTrail to write."
  resolution: "Add CloudTrail service principal to bucket policy with PutObject and GetBucketAcl."

- symptoms: "Trail name already exists"
  diagnosis: "Trail with same name exists in the account."
  resolution: "Use a different trail name or delete the existing trail."

- symptoms: "KMS key access denied"
  diagnosis: "KMS key policy doesn't allow CloudTrail to encrypt."
  resolution: "Add CloudTrail service principal to KMS key policy with GenerateDataKey permission."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create S3 bucket with correct policy | GREEN | New resource creation |
| Fix IAM permissions for trail creation | YELLOW | IAM changes have broad impact potential |
| Create trail | GREEN | New resource creation, enables logging |
| Start logging | GREEN | Enables event recording |
| Add KMS encryption | GREEN | Additive security enhancement |

## Escalation Conditions

- Trail creation blocked by organizational policy — escalate to security/compliance team
- S3 bucket policy changes required in a shared bucket — escalate to bucket owner
- KMS key policy changes required — escalate to key administrator
- Organization trail creation — escalate to management account administrator
- Trail configuration changes in production — escalate to security operations team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail describe-trails` | HIGH | Exposes trail configuration, S3 bucket names, and KMS key ARNs |
| `aws s3api get-bucket-policy` | HIGH | Exposes bucket access policy with account IDs and principals |
| `aws iam simulate-principal-policy` | MEDIUM | Exposes IAM permission evaluation results |
| `aws cloudtrail lookup-events` | HIGH | Exposes API call history with principals and parameters |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** create trails that log to publicly accessible S3 buckets

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created S3 bucket | Delete the bucket if empty and not needed: `aws s3api delete-bucket` |
| Created trail | Delete the trail: `aws cloudtrail delete-trail --name <name>` |
| Started logging | Stop logging: `aws cloudtrail stop-logging --name <name>` (use with caution) |
| Added KMS encryption | Update trail to remove KMS key (reverts to SSE-S3) |

## Output Format

```yaml
root_cause: "trail_creation — <specific_cause>"
evidence:
  - type: bucket_policy
    content: "<S3 bucket policy>"
  - type: iam_permissions
    content: "<IAM permission check>"
severity: HIGH
mitigation:
  immediate: "Fix the creation blocker"
  long_term: "Automate trail creation with IaC"
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
