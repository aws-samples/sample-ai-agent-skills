---
title: "E1 — S3 Package Storage"
description: "Diagnose SSM Distributor S3 package storage and access issues"
status: active
severity: HIGH
triggers: ["s3 access denied", "package download failed", "artifact not found", "s3 bucket error"]
owner: devops-agent
objective: "Resolve S3 storage and access issues for Distributor packages"
context: "Custom packages store artifacts in S3. The SSM Agent downloads artifacts directly from S3 using the instance profile's IAM role."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket exists and is in the correct region: `aws s3api head-bucket --bucket <bucket-name>`
- Check package artifacts exist: `aws s3 ls s3://<bucket>/packages/<package-name>/ --recursive`
- Verify instance profile has s3:GetObject permission: `aws iam get-instance-profile --instance-profile-name <profile-name>`

SHOULD:
- Check S3 bucket policy for deny statements: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify VPC endpoint for S3 if instances are in a private subnet: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.s3`
- Check S3 bucket encryption settings: `aws s3api get-bucket-encryption --bucket <bucket-name>`

MAY:
- Review S3 access logs for denied requests
- Check if S3 Block Public Access settings affect access

## Phase 2 — Remediate

MUST:
- Grant s3:GetObject permission to the instance profile IAM role for the package bucket
- Ensure S3 bucket is in the same region as target instances
- Fix bucket policy deny statements that block SSM Agent access

SHOULD:
- Configure VPC endpoint for S3 if instances lack internet access
- Use S3 bucket policies with least-privilege access

MAY:
- Enable S3 access logging for audit purposes
- Use S3 versioning for package artifact management

## Common Issues

- symptoms: "Installation fails with 'Access Denied' when downloading from S3"
  diagnosis: "Instance profile IAM role lacks s3:GetObject permission for the package bucket."
  resolution: "Add s3:GetObject permission for the specific bucket and prefix to the instance profile role."

- symptoms: "Installation fails with 'NoSuchKey' error"
  diagnosis: "S3 artifact path in manifest does not match actual S3 object key."
  resolution: "Verify S3 object keys match the paths referenced in manifest.json. Check for case sensitivity."

## Safety Ratings

```
safety_ratings:
  - "s3api head-bucket: GREEN — read-only bucket existence check"
  - "s3 ls: GREEN — read-only object listing"
  - "s3api get-bucket-policy: GREEN — read-only policy review"
  - "iam put-role-policy: RED — modifies IAM permissions"
```

## Escalation Conditions

- S3 bucket policy is managed by a central security team and cannot be modified
- VPC endpoint configuration requires network team involvement
- S3 bucket is in a different account requiring cross-account access setup
- KMS encryption on S3 bucket requires key policy changes

## Data Sensitivity

- `aws s3 ls`: LOW — returns object keys and sizes
- `aws s3api get-bucket-policy`: MEDIUM — reveals access control configuration
- `aws iam get-instance-profile`: MEDIUM — reveals IAM role configuration
- S3 access logs: MEDIUM — reveal access patterns and principal identities

## Prohibited Actions

- NEVER make S3 buckets public to resolve access issues
- NEVER use wildcard (*) resource in IAM policies for S3 access
- NEVER disable S3 encryption to troubleshoot access issues
- NEVER store package artifacts in S3 buckets without versioning enabled

## Phase 3 — Rollback

1. If IAM permissions were broadened, revert to the original policy: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://original-policy.json`
2. If bucket policy was modified, restore the original: `aws s3api put-bucket-policy --bucket <bucket> --policy file://original-bucket-policy.json`
3. If VPC endpoint was created for troubleshooting, evaluate whether to keep or remove it
4. Verify package installation works with the corrected permissions
5. Document S3 access requirements for the package

## Output Format

```yaml
root_cause: "s3_storage — <specific_cause>"
evidence:
  - type: s3_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 access permissions"
  long_term: "Standardize S3 bucket configuration for packages"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
