---
title: "D1 — Output Configuration Errors"
description: "Diagnose output configuration failures including S3 permissions, path issues, and format errors"
status: active
severity: MEDIUM
triggers: ["output error", "cannot write results", "S3 output failed", "output configuration invalid"]
owner: devops-agent
objective: "Resolve output configuration errors for matching and ID mapping workflows"
context: "Output configuration determines where results are written. S3 output requires proper bucket permissions. The service role needs write access to the output location. Output failures occur after processing completes — data is processed but results cannot be saved."
---

## Phase 1 — Triage

MUST:
- Get workflow output configuration: `aws entityresolution get-matching-workflow --workflow-name <workflow-name>`
- Check S3 bucket permissions: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify service role has S3 write permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:PutObject --resource-arns arn:aws:s3:::<bucket>/*`

SHOULD:
- Check S3 bucket exists and is accessible
- Verify KMS key permissions if server-side encryption is configured
- Review the output path for correctness

MAY:
- Check S3 bucket versioning and lifecycle policies
- Verify no S3 bucket policy denies the service role
- Test write access manually: `aws s3 cp test.txt s3://<bucket>/<path>/test.txt`

## Phase 2 — Remediate

MUST:
- Grant the service role s3:PutObject permission on the output bucket/path
- Fix the output S3 path in the workflow configuration
- Ensure KMS key allows the service role to encrypt if SSE-KMS is used

SHOULD:
- Use a dedicated output bucket for Entity Resolution results
- Configure appropriate S3 lifecycle policies for output data
- Test output configuration before running large jobs

MAY:
- Set up S3 event notifications for output file creation
- Implement output data validation after job completion

## Safety Ratings

```
safety_ratings:
  - "get-matching-workflow: GREEN — read-only workflow configuration"
  - "s3api get-bucket-policy: GREEN — read-only bucket policy"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "s3 cp (test write): YELLOW — writes test file to bucket"
```

## Escalation Conditions

- Output failure occurs after hours of processing, wasting compute
- S3 bucket policy changes require bucket owner approval
- KMS key policy changes require security team approval
- Output location is in a different account requiring cross-account setup

## Data Sensitivity

- `aws entityresolution get-matching-workflow`: LOW — workflow configuration
- `aws s3api get-bucket-policy`: MEDIUM — reveals access policies
- `aws iam simulate-principal-policy`: LOW — returns allow/deny results
- Output data: HIGH — contains matched records with PII

## Prohibited Actions

- NEVER write output to publicly accessible S3 buckets
- NEVER disable S3 encryption for output containing PII
- NEVER grant s3:* permissions to the service role — use specific actions
- NEVER delete output data without verifying downstream consumers have processed it

## Phase 3 — Rollback

1. If output was written to the wrong location, move or delete the files
2. If S3 permissions were over-granted, revert to previous policy
3. If KMS key policy was modified, restore previous configuration
4. Verify the workflow output configuration is correct
5. Re-run the job after fixing output configuration

## Common Issues

- symptoms: "Job completes processing but fails at output stage"
  diagnosis: "The service role lacks s3:PutObject permission on the output bucket."
  resolution: "Grant s3:PutObject and s3:GetBucketLocation permissions to the service role for the output bucket."

- symptoms: "Output fails with KMS AccessDeniedException"
  diagnosis: "The S3 bucket uses SSE-KMS encryption and the service role cannot use the KMS key."
  resolution: "Add the service role to the KMS key policy with kms:GenerateDataKey and kms:Decrypt permissions."

- symptoms: "Output path does not exist or is incorrect"
  diagnosis: "The S3 prefix in the output configuration is wrong or the bucket does not exist."
  resolution: "Verify the S3 bucket name and prefix. S3 creates prefixes automatically — the bucket must exist."

## Output Format

```yaml
root_cause: "output_configuration — <specific_cause>"
evidence:
  - type: output_error
    content: "<S3 permission or configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix output permissions"
  long_term: "Implement output validation and monitoring"
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
