---
title: "A2 — Data Repository Association Issues"
description: "Diagnose Lustre data repository association (S3 link) problems"
status: active
severity: HIGH
triggers:
  - "data repository association"
  - "DRA failed"
  - "S3 link broken"
  - "import from S3"
  - "export to S3"
  - "auto-import not working"
owner: devops-agent
objective: "Identify and resolve issues with Lustre data repository associations linking to S3"
context: "Lustre DRA links the file system to an S3 bucket for lazy-loading data and exporting changes. DRA failures are commonly caused by S3 bucket policy restrictions, IAM role permissions, S3 bucket region mismatch, or incorrect path configuration."
---

## Phase 1 — Triage

MUST:
- List DRA associations: `aws fsx describe-data-repository-associations --filters Name=file-system-id,Values=<fs-id>`
- Check DRA Lifecycle status and FailureDetails
- Verify S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <bucket-name>`
- Check the IAM role for S3 access: `aws iam get-role --role-name <fsx-role>`

SHOULD:
- Verify S3 bucket policy allows FSx access: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Check S3 bucket is in the same region as the file system
- List recent data repository tasks: `aws fsx describe-data-repository-tasks --filters Name=file-system-id,Values=<fs-id>`
- Check CloudTrail for DRA-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`

MAY:
- Verify auto-import configuration on the DRA
- Check S3 event notification configuration: `aws s3api get-bucket-notification-configuration --bucket <bucket-name>`
- Monitor import/export task progress via CloudWatch metrics

## Phase 2 — Remediate

MUST:
- Fix S3 bucket policy to allow FSx service access
- Ensure IAM role has s3:GetObject, s3:PutObject, s3:ListBucket permissions
- Verify S3 bucket and FSx file system are in the same AWS region

SHOULD:
- Enable auto-import for automatic metadata synchronization with S3
- Use data repository tasks for bulk import/export operations
- Monitor DRA health via CloudWatch metrics

MAY:
- Configure S3 lifecycle policies to complement FSx data management
- Set up CloudWatch alarms for DRA task failures

## Common Issues

- symptoms: "DRA status FAILED or MISCONFIGURED"
  diagnosis: "S3 bucket policy blocks FSx access or IAM role lacks S3 permissions."
  resolution: "Update S3 bucket policy to allow fsx.amazonaws.com. Verify IAM role has S3 read/write permissions."

- symptoms: "Auto-import not reflecting S3 changes"
  diagnosis: "Auto-import is not enabled on the DRA or S3 event notifications are misconfigured."
  resolution: "Enable auto-import on the DRA. FSx manages S3 event notifications automatically."

- symptoms: "Export task fails with access denied"
  diagnosis: "IAM role lacks s3:PutObject permission on the target S3 prefix."
  resolution: "Update IAM role to include s3:PutObject and s3:PutObjectAcl for the S3 bucket/prefix."

## Output Format

```yaml
root_cause: "dra_issue — <specific_cause>"
evidence:
  - type: dra_status
    content: "<describe-data-repository-associations output>"
  - type: s3_bucket
    content: "<bucket policy and accessibility>"
  - type: iam_role
    content: "<role permissions for S3 access>"
severity: HIGH
mitigation:
  immediate: "Fix S3 bucket policy and IAM permissions, recreate DRA if needed"
  long_term: "Standardize DRA configuration with IaC templates"
```


---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — FSx provides shared file storage for compute workloads, databases, and analytics |
| Data Risk | CRITICAL — File systems contain production data, application state, and potentially irreplaceable datasets |
| Recovery Complexity | HIGH — File system recreation requires backup restoration and client remounting across all consumers |
| Blast Radius | HIGH — File system outages affect all connected compute instances and dependent applications |

## Escalation Conditions

- File system enters FAILED or MISCONFIGURED state with no clear remediation path
- Multi-AZ failover not completing or standby file server unavailable during AZ event
- Data corruption detected on file system volumes or inconsistent backup state
- KMS key disabled or deleted causing file system inaccessibility
- Storage capacity at 100% utilization with no ability to scale (quota limits reached)
- Backup or restore operations failing on business-critical file systems

## Data Sensitivity

- File systems may contain regulated data (financial records, healthcare data, PII)
- Backup contents mirror production data sensitivity classifications
- Active Directory integration credentials reveal domain architecture and trust relationships
- NFS export policies and NTFS ACLs expose organizational access control structure
- ONTAP SVM configurations reveal multi-tenant data isolation boundaries

## Prohibited Actions

- NEVER suggest deleting file systems without confirming backups exist and are restorable
- NEVER suggest disabling encryption or using unencrypted file systems for any workload
- NEVER suggest reducing storage capacity — this operation is irreversible on all FSx types
- NEVER modify Active Directory integration on production Windows FSx without maintenance window
- NEVER remove or modify KMS key policies without verifying all dependent file systems
- NEVER change security group rules that could disconnect active file system clients

## Phase 3 — Rollback

- If file system configuration was changed: revert settings and verify client connectivity is restored
- If storage scaling failed: wait for pending operations to complete, verify file system returns to AVAILABLE
- If backup was deleted: check AWS Backup vault for retained copies, recreate from most recent available backup
- If security group was modified: restore previous rules immediately and verify all client mounts are functional
- If KMS key was affected: re-enable key or cancel deletion, verify file system recovers automatically
- Document all rollback actions with file system ID, affected clients, and data integrity verification steps

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest removing AD integration without review"
  - "NEVER suggest public subnet placement"
