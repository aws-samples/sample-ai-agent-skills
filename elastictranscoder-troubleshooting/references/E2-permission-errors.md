---
title: "E2 — Permission Errors"
description: "Diagnose Elastic Transcoder IAM permission errors"
status: active
severity: HIGH
triggers: ["AccessDenied", "permission denied", "IAM role error", "cannot access S3"]
owner: devops-agent
objective: "Resolve IAM permission issues affecting pipeline operations, job execution, and S3 access"
context: "Elastic Transcoder uses an IAM service role for S3 and SNS access. The role must trust elastictranscoder.amazonaws.com and have specific S3/SNS permissions."
---

## Phase 1 — Triage

MUST:
- Read pipeline to get IAM role ARN: `aws elastictranscoder read-pipeline --id <pipeline-id>`
- Verify IAM role exists: `aws iam get-role --role-name <role-name>`
- Check role trust policy includes elastictranscoder.amazonaws.com
- List attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- List inline policies: `aws iam list-role-policies --role-name <role-name>`

SHOULD:
- Verify s3:GetObject on input bucket, s3:PutObject on output bucket, s3:PutObject on thumbnail bucket
- Check S3 bucket policies for explicit denies: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify sns:Publish permission if notifications are configured

MAY:
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*`
- Check for SCPs that may restrict Elastic Transcoder operations

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policy to include elastictranscoder.amazonaws.com
- Add required S3 permissions (GetObject for input, PutObject for output and thumbnails)
- Remove explicit deny statements in S3 bucket policies that block the service role

SHOULD:
- Use least-privilege permissions scoped to specific buckets
- Add ListBucket permission for better error messages on missing files

MAY:
- Create a dedicated IAM role per pipeline for isolation
- Set up IAM Access Analyzer to monitor the role permissions

## Safety Ratings

```
safety_ratings:
  - "iam get-role: GREEN — read-only role inspection"
  - "iam list-attached-role-policies: GREEN — read-only policy listing"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "iam put-role-policy: RED — modifies IAM permissions"
```

## Escalation Conditions

- IAM role modifications are blocked by organizational SCPs
- S3 bucket policy is managed by another team and cannot be modified
- Permission errors persist after all IAM fixes are applied
- Cross-account S3 access is required and bucket owner denies access

## Data Sensitivity

- `aws iam get-role`: LOW — returns role metadata and trust policy
- `aws iam list-attached-role-policies`: LOW — returns policy names
- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access controls
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes

## Prohibited Actions

- NEVER grant s3:* or * permissions to the Elastic Transcoder role
- NEVER modify IAM roles used by other services without verifying impact
- NEVER remove S3 bucket policies without understanding all access patterns
- NEVER use the root account credentials for Elastic Transcoder operations

## Phase 3 — Rollback

1. If IAM role policy was modified, revert to the previous policy version
2. If S3 bucket policy was changed, restore the original policy: `aws s3api put-bucket-policy --bucket <bucket> --policy file://original-policy.json`
3. If a new IAM role was created, delete it if not needed: `aws iam delete-role --role-name <role-name>` (after detaching policies)
4. Verify pipeline still functions correctly after permission changes
5. Document all IAM changes made during troubleshooting for audit trail

## Common Issues

- symptoms: "Job fails with AccessDenied on GetObject"
  diagnosis: "IAM role lacks s3:GetObject permission on the input bucket."
  resolution: "Add s3:GetObject permission for arn:aws:s3:::<input-bucket>/* to the role policy."

- symptoms: "Pipeline creation fails with role assumption error"
  diagnosis: "IAM role trust policy does not include elastictranscoder.amazonaws.com."
  resolution: "Update trust policy: add elastictranscoder.amazonaws.com as a trusted service principal."

## Output Format

```yaml
root_cause: "permission_error — <specific_cause>"
evidence:
  - type: iam_issue
    content: "<role details and missing permissions>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement least-privilege role with IaC"
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
