---
title: "D1 — S3 Backend Issues"
description: "Diagnose S3 storage backend access, permission, and configuration problems"
status: active
severity: HIGH
triggers:
  - "S3 access denied"
  - "cannot upload to S3"
  - "S3 permission"
  - "bucket not found"
  - "S3 backend error"
  - "file not visible"
owner: devops-agent
objective: "Resolve S3 backend access and configuration issues for Transfer Family users"
context: "Transfer Family accesses S3 using the IAM role assigned to each user. The role must have S3 permissions, the bucket policy must not deny access, and session policies (if configured) must also allow the operations. S3 bucket versioning, encryption, and object lock can also affect file operations."
---

## Phase 1 — Triage

MUST:
- Check user's IAM role: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.Role'`
- Verify role has S3 permissions: `aws iam list-attached-role-policies --role-name <role-name>` and `aws iam list-role-policies --role-name <role-name>`
- Check the S3 bucket exists and is in the same region: `aws s3api head-bucket --bucket <bucket-name>`
- Check bucket policy for explicit denies: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Check if session policy restricts access: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.Policy'`

SHOULD:
- Check S3 Block Public Access settings: `aws s3api get-public-access-block --bucket <bucket-name>`
- Verify bucket encryption configuration: `aws s3api get-bucket-encryption --bucket <bucket-name>`
- If SSE-KMS, verify the IAM role has kms:Decrypt and kms:GenerateDataKey permissions
- Check if the home directory prefix exists: `aws s3 ls s3://<bucket>/<prefix>/`

MAY:
- Check S3 access logs for denied requests
- Verify no S3 Object Lock or Legal Hold is preventing operations
- Check if bucket versioning affects file visibility

## Phase 2 — Remediate

MUST:
- Grant the IAM role s3:ListBucket, s3:GetObject, s3:PutObject, s3:DeleteObject on the bucket and objects
- Remove or modify bucket policy explicit denies that block the Transfer Family role
- Ensure session policy (if used) includes all required S3 actions
- For SSE-KMS buckets, add kms:Decrypt, kms:GenerateDataKey, kms:DescribeKey to the role

SHOULD:
- Use session policies with ${transfer:UserName} variables to scope access per user
- Create the home directory prefix in S3 if it doesn't exist (S3 doesn't require it, but some clients expect it)
- Use S3 bucket policies to restrict access to Transfer Family roles only

MAY:
- Enable S3 server access logging for audit trails
- Implement S3 lifecycle policies for uploaded files

## Common Issues

- symptoms: "Access denied when uploading files"
  diagnosis: "IAM role lacks s3:PutObject permission, or bucket policy has an explicit deny."
  resolution: "Add s3:PutObject to the role policy. Check bucket policy for Deny statements."

- symptoms: "Files uploaded but not visible in S3 console"
  diagnosis: "Files uploaded to a different prefix than expected due to directory mapping misconfiguration."
  resolution: "Check HomeDirectory and HomeDirectoryMappings for the correct S3 prefix."

- symptoms: "Access denied only for specific files"
  diagnosis: "Bucket uses SSE-KMS encryption and the IAM role lacks KMS permissions."
  resolution: "Add kms:Decrypt and kms:GenerateDataKey permissions for the KMS key to the role."

## Output Format

```yaml
root_cause: "s3_backend — <specific_cause>"
evidence:
  - type: iam_role
    content: "<role policies and permissions>"
  - type: bucket_policy
    content: "<bucket policy>"
  - type: session_policy
    content: "<session policy if configured>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role permissions or bucket policy"
  long_term: "Implement least-privilege policies with session policy scoping"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
