---
title: "C1 — S3 Location Configuration Issues"
description: "Diagnose S3 location creation and access problems in DataSync"
status: active
severity: HIGH
triggers:
  - "S3 location error"
  - "cannot access S3 bucket"
  - "S3 permission denied"
  - "location creation failed S3"
owner: devops-agent
objective: "Identify and resolve S3 location configuration issues in DataSync"
context: "S3 locations require a properly configured IAM role with S3 permissions. The role must trust the DataSync service principal. Common issues include incorrect IAM role, missing S3 permissions, bucket policy restrictions, KMS key access for encrypted buckets, and incorrect subdirectory configuration."
---

## Phase 1 — Triage

MUST:
- Check S3 location configuration: `aws datasync describe-location-s3 --location-arn <location-arn>`
- Verify IAM role trust policy allows DataSync: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check IAM role permissions: `aws iam list-attached-role-policies --role-name <role-name>` and `aws iam list-role-policies --role-name <role-name>`
- Verify S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <bucket-name>`

SHOULD:
- Check bucket policy for restrictions: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify KMS key access if bucket uses SSE-KMS: `aws kms describe-key --key-id <key-id>`
- Check if bucket has Object Lock or versioning that may affect writes

MAY:
- Test IAM role permissions with policy simulator
- Check S3 bucket region matches expectations

## Phase 2 — Remediate

MUST:
- Ensure IAM role trust policy includes: `"Service": "datasync.amazonaws.com"`
- Grant required S3 permissions: s3:GetBucketLocation, s3:ListBucket, s3:GetObject, s3:PutObject, s3:DeleteObject
- For KMS-encrypted buckets, grant kms:Decrypt and kms:GenerateDataKey on the KMS key

SHOULD:
- Create S3 location with correct configuration:
  `aws datasync create-location-s3 --s3-bucket-arn arn:aws:s3:::<bucket> --s3-config BucketAccessRoleArn=<role-arn> --subdirectory /<prefix>/`

MAY:
- Use S3 storage class options (STANDARD, INTELLIGENT_TIERING, etc.) in location config
- Configure S3 bucket lifecycle policies for transferred data

## Common Issues

- symptoms: "Access Denied when task accesses S3"
  diagnosis: "IAM role missing required S3 permissions or trust policy."
  resolution: "Update IAM role with S3 permissions and DataSync trust policy."

- symptoms: "KMS error when writing to encrypted bucket"
  diagnosis: "IAM role lacks KMS permissions for the bucket's encryption key."
  resolution: "Grant kms:Decrypt and kms:GenerateDataKey to the DataSync IAM role."

- symptoms: "No files transferred but task succeeds"
  diagnosis: "Subdirectory path is incorrect or empty."
  resolution: "Verify subdirectory parameter matches the S3 prefix containing files."

## Output Format

```yaml
root_cause: "s3_location_issue — <specific_cause>"
evidence:
  - type: location_config
    content: "<S3 location configuration>"
  - type: iam_role
    content: "<IAM role trust and permissions>"
  - type: bucket_access
    content: "<bucket accessibility and policy>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role permissions and location configuration"
  long_term: "Use IaC to manage DataSync locations with correct IAM roles"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — S3 locations may contain sensitive data; IAM roles have broad access |
| Blast Radius | MEDIUM — Incorrect location config causes task failures |
| Rollback Complexity | LOW — Recreate location with correct configuration |
| Compliance Scope | HIGH — S3 data access must comply with data governance policies |

## Escalation Conditions

- Escalate to security if IAM role has overly broad S3 permissions
- Escalate to data owners if KMS key access issues affect encrypted data
- Escalate to compliance if S3 bucket lacks required encryption or access controls

## Data Sensitivity

- **IAM role** for S3 access has read/write to potentially sensitive data — audit regularly
- **KMS key access** grants decryption capability — restrict to required roles only
- **S3 bucket contents** may be highly sensitive — verify bucket policies

## PROHIBITED Actions

- Do NOT grant s3:* permissions to the DataSync IAM role — use least-privilege
- Do NOT create S3 locations pointing to buckets without encryption
- Do NOT share IAM role ARNs or KMS key IDs outside authorized teams
- Do NOT use the same IAM role for multiple unrelated DataSync locations

## Phase 3 — Rollback

- Delete the misconfigured location and recreate with correct settings
- Revert IAM role permissions if they were broadened during troubleshooting
- Restore KMS key policy if it was modified
- Verify the new location works with a test task execution

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
