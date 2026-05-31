---
title: "E1 — S3 Data Access"
description: "Diagnose S3 data access issues for training data, seed audiences, and output"
status: active
severity: HIGH
triggers: ["S3 access denied", "cannot read data", "bucket not found", "data access error"]
owner: devops-agent
objective: "Resolve S3 data access issues for Clean Rooms ML operations"
context: "Clean Rooms ML reads training data and seed audiences from S3 and writes output to S3. All buckets must be in the same region as the collaboration. Service roles need proper S3 and KMS permissions."
---

## Phase 1 — Triage

MUST:
- Verify bucket exists and region matches collaboration: `aws s3api get-bucket-location --bucket <bucket>`
- Check bucket policy for service principal access: `aws s3api get-bucket-policy --bucket <bucket>`
- Verify service role S3 permissions: `aws iam get-role-policy --role-name <role> --policy-name <policy>`

SHOULD:
- Check S3 Block Public Access settings: `aws s3api get-public-access-block --bucket <bucket>`
- Verify KMS key policy if bucket uses SSE-KMS: `aws kms get-key-policy --key-id <key-id> --policy-name default`

MAY:
- Test access with simulate-principal-policy: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*`
- Check VPC endpoint policies if bucket is accessed via VPC endpoint

## Phase 2 — Remediate

MUST:
- Add s3:GetObject, s3:GetBucketLocation, s3:ListBucket to the service role for input buckets
- Add s3:PutObject to the service role for output buckets
- Ensure bucket is in the same AWS region as the collaboration

SHOULD:
- Add kms:Decrypt for input buckets and kms:GenerateDataKey for output buckets
- Update bucket policy to explicitly allow the Clean Rooms ML service role

MAY:
- Create dedicated S3 buckets with pre-configured permissions for Clean Rooms ML
- Enable S3 access logging for debugging future access issues

## Common Issues

- symptoms: "AccessDenied when training reads from S3"
  diagnosis: "Service role lacks s3:GetObject on the training data bucket."
  resolution: "Add s3:GetObject and s3:ListBucket to the service role. Verify bucket policy does not have explicit deny."

- symptoms: "S3 bucket not found error"
  diagnosis: "Bucket is in a different region than the collaboration."
  resolution: "Verify bucket region with get-bucket-location. Move data to a bucket in the collaboration region."

## Safety Ratings

```
safety_ratings:
  - "s3api get-bucket-location: GREEN — read-only metadata"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "s3api put-bucket-policy: YELLOW — modifies bucket access"
  - "iam put-role-policy: YELLOW — modifies IAM role permissions"
```

## Escalation Conditions

- S3 bucket is owned by another AWS account and requires cross-account policy changes
- Bucket policy has explicit deny statements from SCP or organization policies
- KMS key is in another account and requires cross-account key policy changes
- VPC endpoint policy blocks S3 access and requires network team involvement

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals access policy details
- `aws iam get-role-policy`: MEDIUM — reveals IAM permissions
- `aws kms get-key-policy`: MEDIUM — reveals key access policy
- `aws s3 ls s3://<bucket>/`: MEDIUM — reveals data file names

## Prohibited Actions

- NEVER grant s3:* (wildcard) permissions — use least privilege
- NEVER modify bucket policies to allow public access
- NEVER change S3 bucket ownership or ACLs without data owner approval
- NEVER access data objects directly — only verify access permissions

## Phase 3 — Rollback

1. If bucket policy was changed incorrectly, restore: `aws s3api put-bucket-policy --bucket <bucket> --policy file://previous-policy.json`
2. If IAM role policy was modified, restore previous version: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://previous-policy.json`
3. If KMS key policy was changed, restore: `aws kms put-key-policy --key-id <id> --policy-name default --policy file://previous-key-policy.json`
4. Verify no unauthorized access occurred during the misconfiguration window
5. Document all permission changes for audit trail

## Output Format

```yaml
root_cause: "s3_data_access — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details from S3 or IAM inspection>"
severity: HIGH
mitigation:
  immediate: "Fix S3 and IAM permissions"
  long_term: "Use IaC to manage S3 bucket policies for Clean Rooms ML"
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
