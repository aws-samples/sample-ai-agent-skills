---
title: "E1 — S3 Permissions"
description: "Diagnose S3 permission issues for datasets, training, and model artifacts"
status: active
severity: HIGH
triggers: ["S3 access denied", "cannot read images", "bucket not found", "permission error"]
owner: devops-agent
objective: "Resolve S3 permission issues for Lookout for Vision operations"
context: "Lookout for Vision reads training images from S3, stores model artifacts in S3, and writes packaging output to S3. All buckets must be in the same region as the project."
---

## Phase 1 — Triage

MUST:
- Verify bucket exists and region matches project: `aws s3api get-bucket-location --bucket <bucket>`
- Check bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Verify service role S3 permissions: `aws iam list-attached-role-policies --role-name <role>`

SHOULD:
- Check S3 Block Public Access settings: `aws s3api get-public-access-block --bucket <bucket>`
- Verify KMS key policy if bucket uses SSE-KMS: `aws kms get-key-policy --key-id <key-id> --policy-name default`

MAY:
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*`
- Check VPC endpoint policies if applicable

## Phase 2 — Remediate

MUST:
- Add s3:GetObject, s3:GetBucketLocation, s3:ListBucket for input buckets
- Add s3:PutObject for output buckets (model artifacts, packaging)
- Ensure bucket is in the same region as the Lookout for Vision project

SHOULD:
- Add kms:Decrypt for input and kms:GenerateDataKey for output if using SSE-KMS
- Scope permissions to specific bucket ARNs (least privilege)

MAY:
- Create dedicated S3 buckets for Lookout for Vision with pre-configured permissions
- Enable S3 access logging for debugging

## Common Issues

- symptoms: "Dataset import fails with AccessDenied"
  diagnosis: "Service role lacks s3:GetObject on the image bucket."
  resolution: "Add s3:GetObject and s3:ListBucket to the service role for the image bucket ARN."

- symptoms: "S3 bucket not found during dataset import"
  diagnosis: "Bucket is in a different region than the project."
  resolution: "Verify bucket region. Move images to a bucket in the project region."

## Safety Ratings

```
safety_ratings:
  - "s3api get-bucket-location: GREEN — read-only metadata"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "s3api put-bucket-policy: YELLOW — modifies bucket access"
  - "iam put-role-policy: YELLOW — modifies IAM permissions"
```

## Escalation Conditions

- S3 bucket is in a different account requiring cross-account access
- Bucket policy has explicit deny from SCP or organization policies
- KMS key is managed by security team requiring approval
- VPC endpoint policy blocks S3 access

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals access policy
- `aws iam list-attached-role-policies`: LOW — returns policy names
- `aws kms get-key-policy`: MEDIUM — reveals key access policy
- Training images in S3: HIGH — may contain proprietary images

## Prohibited Actions

- NEVER grant s3:* wildcard permissions
- NEVER modify bucket policies to allow public access
- NEVER change bucket ownership without data owner approval
- NEVER access image content directly — only verify permissions

## Phase 3 — Rollback

1. If bucket policy was changed incorrectly, restore: `aws s3api put-bucket-policy --bucket <bucket> --policy file://previous-policy.json`
2. If IAM role was modified, restore previous policy version
3. If KMS key policy was changed, restore from backup
4. Verify no unauthorized access occurred during misconfiguration
5. Document all permission changes for audit

## Output Format

```yaml
root_cause: "s3_permissions — <specific_cause>"
evidence:
  - type: permission_error
    content: "<S3 policy, IAM role, KMS key details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 and IAM permissions"
  long_term: "Use IaC to manage S3 bucket policies"
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
