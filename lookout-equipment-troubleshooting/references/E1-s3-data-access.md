---
title: "E1 — S3 Data Access"
description: "Diagnose Amazon Lookout for Equipment S3 data access issues"
status: active
severity: HIGH
triggers: ["cannot access S3 data", "S3 permission denied", "data not found", "bucket access error"]
owner: devops-agent
objective: "Resolve S3 data access issues for Lookout for Equipment operations"
context: "Lookout for Equipment reads sensor data from S3 for ingestion and inference, and writes results to S3. The S3 bucket must be in the same region as the service."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket exists and is in the same region: `aws s3api get-bucket-location --bucket <bucket>`
- Check IAM role permissions: `aws iam get-role-policy --role-name <role> --policy-name <policy>`
- Verify S3 bucket policy does not deny access: `aws s3api get-bucket-policy --bucket <bucket>`
- Check if S3 Block Public Access is interfering: `aws s3api get-public-access-block --bucket <bucket>`

SHOULD:
- Test role access with simulation: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject s3:PutObject --resource-arns arn:aws:s3:::<bucket>/*`
- Verify the S3 path prefix exists and contains expected files: `aws s3 ls s3://<bucket>/<prefix>/`
- Check for S3 VPC endpoint policies if running in a VPC

MAY:
- Review S3 access logs for denied requests
- Check AWS CloudTrail for S3 access denied events
- Verify KMS key permissions if the bucket uses SSE-KMS encryption

## Phase 2 — Remediate

MUST:
- Ensure IAM role has s3:GetObject, s3:ListBucket, s3:GetBucketLocation on the data bucket
- For inference output: add s3:PutObject permission on the output path
- Add lookoutequipment.amazonaws.com to the IAM role trust policy
- Ensure S3 bucket is in the same AWS region as Lookout for Equipment resources

SHOULD:
- If bucket uses SSE-KMS, grant kms:Decrypt and kms:GenerateDataKey to the service role
- Remove overly restrictive bucket policies that block the service role
- Use S3 bucket policies to explicitly allow the Lookout for Equipment service role

MAY:
- Enable S3 server access logging to diagnose intermittent access issues
- Configure VPC endpoint policies to allow Lookout for Equipment access

## Common Issues

- symptoms: "Access denied when starting data ingestion job"
  diagnosis: "The IAM role trust policy does not include lookoutequipment.amazonaws.com as a trusted principal."
  resolution: "Update the role trust policy: add lookoutequipment.amazonaws.com as an allowed AssumeRole principal."

- symptoms: "Inference scheduler cannot write results to S3"
  diagnosis: "The IAM role has read permissions but lacks s3:PutObject on the output path."
  resolution: "Add s3:PutObject permission for the output S3 path to the IAM role policy."

- symptoms: "Access works for some operations but not others"
  diagnosis: "S3 bucket uses SSE-KMS encryption and the role lacks KMS permissions."
  resolution: "Grant kms:Decrypt for reading and kms:GenerateDataKey for writing to the KMS key used by the bucket."

## Safety Ratings

```
safety_ratings:
  - "s3api get-bucket-location: GREEN — read-only bucket metadata"
  - "iam get-role-policy: GREEN — read-only policy inspection"
  - "s3api get-bucket-policy: GREEN — read-only policy inspection"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "Modifying IAM role policy: YELLOW — changes access permissions"
  - "Modifying S3 bucket policy: RED — affects all access to the bucket"
```

## Escalation Conditions

- S3 access denied persists after IAM and bucket policy fixes
- Bucket is owned by a different account and cross-account access is needed
- VPC endpoint policies are managed by a separate network team
- KMS key policy changes require key administrator approval

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals access control configuration
- `aws iam get-role-policy`: MEDIUM — reveals IAM permissions
- `aws s3 ls`: MEDIUM — reveals S3 object paths and structure
- S3 access logs: HIGH — contain detailed access records with IP addresses and principals

## Prohibited Actions

- NEVER add s3:* wildcard permissions as a troubleshooting shortcut
- NEVER modify S3 bucket policies without understanding the impact on other consumers
- NEVER disable S3 Block Public Access to resolve Lookout for Equipment access issues
- NEVER share IAM role policies or S3 bucket policies externally without redacting account IDs

## Phase 3 — Rollback

1. If IAM role policy was modified incorrectly, restore from version history: `aws iam list-role-policies --role-name <name>`
2. If S3 bucket policy was changed, restore the previous version from CloudTrail records
3. If KMS key policy was modified, restore using: `aws kms get-key-policy --key-id <id> --policy-name default`
4. Verify all Lookout for Equipment operations (ingestion, inference) work after policy changes
5. Document the correct IAM and S3 configuration for the operations runbook

## Output Format

```yaml
root_cause: "s3_data_access — <specific_cause>"
evidence:
  - type: access_error
    content: "<error details from API or CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role or S3 bucket policy"
  long_term: "Implement least-privilege IAM policies with regular access reviews"
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
