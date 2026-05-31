---
title: "E2 — S3 Permissions"
description: "Diagnose S3 permission issues for DataBrew jobs"
status: active
severity: HIGH
triggers:
  - "S3 access denied"
  - "cannot write to S3"
  - "permission denied on bucket"
owner: devops-agent
objective: "Resolve S3 permission issues for DataBrew input and output"
context: "DataBrew jobs need S3 read access for input and write access for output. Permissions come from the job's IAM role. Bucket policies, KMS encryption, and VPC endpoints can also affect access."
---

## Phase 1 — Triage

MUST:
- Check job IAM role: `aws databrew describe-job --name <job-name> --query 'RoleArn'`
- Check role policies: `aws iam list-attached-role-policies --role-name <role>`
- Check bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Check bucket encryption: `aws s3api get-bucket-encryption --bucket <bucket>`

SHOULD:
- Verify KMS key policy allows the DataBrew role if SSE-KMS is used
- Check for S3 Block Public Access settings
- Verify VPC endpoint policy if using VPC endpoints

## Phase 2 — Remediate

MUST:
- Add S3 permissions to role: s3:GetObject, s3:PutObject, s3:ListBucket, s3:DeleteObject
- If SSE-KMS: add kms:Decrypt, kms:GenerateDataKey to the role
- Ensure bucket policy doesn't explicitly deny the DataBrew role

## Common Issues

- symptoms: "AccessDenied on s3:PutObject"
  diagnosis: "Role lacks write permission or bucket policy denies writes."
  resolution: "Add s3:PutObject to role policy for the output bucket/prefix."

- symptoms: "KMS access denied"
  diagnosis: "Bucket uses SSE-KMS but role lacks KMS permissions."
  resolution: "Add kms:Decrypt and kms:GenerateDataKey for the KMS key to the role."

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete datasets that are referenced by active projects or jobs
- Never modify recipe steps on production jobs without testing in a separate project
- Never change output S3 paths for scheduled jobs without updating downstream consumers
- Never delete or overwrite profile job results used for data quality validation
- Never run untested recipe transformations against production data

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Revert recipe changes by republishing the previous recipe version
- Restore dataset configurations from documented state if connections break
- Re-enable previous schedule settings if job scheduling fails after changes

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
  - "NEVER suggest disabling encryption for Glue jobs"
  - "NEVER suggest overly broad Glue service role"
  - "NEVER suggest public S3 access for data catalog"
