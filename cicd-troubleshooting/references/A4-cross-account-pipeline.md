---
title: "A4 — Cross-Account Pipeline"
description: "Diagnose and resolve cross-account CodePipeline access issues"
status: active
severity: HIGH
triggers:
  - "cross-account"
  - "Access Denied"
  - "KMS key"
  - "assume role failed"
  - "artifact access denied"
owner: devops-agent
objective: "Identify and resolve cross-account access issues in CodePipeline"
context: "Cross-account pipelines allow a pipeline in one AWS account to deploy to another account. This requires a customer-managed KMS key (not the default aws/s3 key), S3 bucket policy for cross-account access, and IAM roles with trust policies in both accounts. Missing any component causes Access Denied errors."
---

## Phase 1 — Triage

MUST:
- Check pipeline artifact store encryption: `aws codepipeline get-pipeline --name <pipeline-name> --query 'pipeline.artifactStore'`
- Verify KMS key exists and check its policy: `aws kms describe-key --key-id <key-id>` and `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check the cross-account IAM role trust policy: `aws iam get-role --role-name <cross-account-role> --query 'Role.AssumeRolePolicyDocument'`
- Check S3 artifact bucket policy: `aws s3api get-bucket-policy --bucket <artifact-bucket>`

SHOULD:
- Verify the pipeline action has the correct roleArn for cross-account: `aws codepipeline get-pipeline --name <pipeline-name>`
- Check CloudTrail in BOTH accounts for AssumeRole and KMS errors
- Verify the cross-account role has permissions for the target service (CodeDeploy, CloudFormation, etc.)

MAY:
- Test cross-account role assumption manually: `aws sts assume-role --role-arn <cross-account-role-arn> --role-session-name test`
- Check KMS key grants if using grants instead of key policy
- Verify VPC endpoint policies if using interface endpoints for S3 or KMS

## Phase 2 — Remediate

MUST:
- If using default aws/s3 key: create a customer-managed KMS key and update the pipeline artifact store
- If KMS key policy missing cross-account access: add `kms:Decrypt`, `kms:DescribeKey`, `kms:GenerateDataKey` for the target account role
- If S3 bucket policy missing cross-account access: add `s3:GetObject`, `s3:PutObject` for the target account role
- If trust policy incorrect: update the cross-account role trust policy to allow the pipeline account

SHOULD:
- Use a dedicated cross-account role with least-privilege permissions
- Document the cross-account setup including all three components (KMS, S3, IAM)
- Test the cross-account access before deploying the pipeline

MAY:
- Use AWS Organizations SCPs to control which accounts can be pipeline targets
- Implement resource policies on the target account's resources for additional security

## Common Issues

- symptoms: "Access Denied when downloading artifacts in target account"
  diagnosis: "KMS key policy does not grant decrypt access to the target account role."
  resolution: "Update KMS key policy to grant kms:Decrypt and kms:DescribeKey to the target account role ARN."

- symptoms: "Access Denied — using default aws/s3 encryption key"
  diagnosis: "Default AWS-managed S3 key cannot be shared across accounts."
  resolution: "Create a customer-managed KMS key, update pipeline artifactStore, and grant cross-account access in key policy."

- symptoms: "AssumeRole failed — not authorized to perform sts:AssumeRole"
  diagnosis: "Cross-account role trust policy does not allow the pipeline account to assume it."
  resolution: "Update trust policy to allow sts:AssumeRole from the pipeline account's pipeline service role."

- symptoms: "Deployment succeeds but cannot read artifacts"
  diagnosis: "S3 bucket policy does not grant cross-account access to the artifact objects."
  resolution: "Add s3:GetObject and s3:GetBucketLocation permissions for the target account role in the bucket policy."

## Output Format

```yaml
root_cause: "Cross-account pipeline failure — <specific_cause>"
evidence:
  - type: kms_key_policy
    content: "<key policy details>"
  - type: s3_bucket_policy
    content: "<bucket policy details>"
  - type: iam_trust_policy
    content: "<trust policy details>"
severity: HIGH
mitigation:
  immediate: "Fix the identified access issue (KMS, S3, or IAM) and retry"
  long_term: "Document cross-account setup, use IaC to manage all three components together"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If using default aws/s3 key: create a customer-managed KMS key | YELLOW |
| If KMS key policy missing cross-account access: add decrypt permissions | RED |
| If S3 bucket policy missing cross-account access: add GetObject/PutObject | RED |
| If trust policy incorrect: update the cross-account role trust policy | RED |
| Use a dedicated cross-account role with least-privilege permissions | GREEN |
| Document the cross-account setup | GREEN |
| Test the cross-account access before deploying | GREEN |
| Use AWS Organizations SCPs to control target accounts | YELLOW |
| Implement resource policies on target account resources | YELLOW |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Cross-account configurations expose trust relationships, KMS key policies, and S3 bucket policies between accounts

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If KMS key policy was modified, restore the previous key policy: `aws kms put-key-policy --key-id <id> --policy-name default --policy <previous-policy>`
2. If S3 bucket policy was modified, restore the previous bucket policy: `aws s3api put-bucket-policy --bucket <bucket> --policy <previous-policy>`
3. If cross-account IAM role trust policy was changed, revert to the previous trust policy
4. If a customer-managed KMS key was created, keep it (deleting may break existing encrypted artifacts)
5. Verify rollback by testing cross-account access from the pipeline

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
