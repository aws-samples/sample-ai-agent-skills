---
title: "F1 — FSx Backup Failures"
description: "Diagnose backup failures across all FSx file system types"
status: active
severity: HIGH
triggers:
  - "fsx backup failed"
  - "backup error"
  - "automatic backup failed"
  - "backup not created"
  - "backup job failure"
owner: devops-agent
objective: "Identify and resolve backup failures on FSx file systems"
context: "FSx supports automatic daily backups and user-initiated backups for all types except Lustre scratch. Backup failures can be caused by KMS key issues, IAM permissions, storage capacity, or file system state. Lustre scratch does NOT support backups."
---

## Phase 1 — Triage

MUST:
- List backups for the file system: `aws fsx describe-backups --filters Name=file-system-id,Values=<fs-id>`
- Check backup Lifecycle status and FailureDetails
- Verify the file system type supports backups (Lustre scratch does NOT)
- Describe the file system: `aws fsx describe-file-systems --file-system-ids <fs-id>`

SHOULD:
- Check KMS key status: `aws kms describe-key --key-id <key-id>`
- Verify automatic backup configuration (window, retention)
- Check CloudTrail for backup-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`
- Verify AWS Backup integration if using centralized backup

MAY:
- Check for concurrent backup operations that may conflict
- Verify service quotas for backup count: `aws service-quotas get-service-quota --service-code fsx --quota-code L-5765E764`
- Monitor backup duration trends

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (KMS, permissions, file system state)
- Verify the file system is in AVAILABLE state before retrying backup
- Ensure KMS key is enabled and accessible

SHOULD:
- Configure automatic backups with appropriate retention period
- Set backup window during low-activity periods
- Integrate with AWS Backup for centralized management

MAY:
- Set up SNS notifications for backup failures
- Create CloudWatch alarms for backup monitoring

## Common Issues

- symptoms: "Backup fails on Lustre file system"
  diagnosis: "The Lustre file system is scratch deployment type which does not support backups."
  resolution: "Backups are only supported for persistent Lustre. Use S3 export for scratch data protection."

- symptoms: "Backup fails with KMS error"
  diagnosis: "The KMS key used for file system encryption is disabled or the backup role lacks access."
  resolution: "Enable the KMS key and verify the FSx service-linked role has kms:CreateGrant permission."

- symptoms: "Automatic backup not being created"
  diagnosis: "Automatic backup retention is set to 0 (disabled) or backup window is misconfigured."
  resolution: "Set AutomaticBackupRetentionDays to a value > 0 and verify the backup window."

## Output Format

```yaml
root_cause: "backup_failure — <specific_cause>"
evidence:
  - type: backup_status
    content: "<describe-backups output with FailureDetails>"
  - type: file_system
    content: "<file system type and backup configuration>"
  - type: kms_key
    content: "<KMS key status and policy>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry backup"
  long_term: "Configure automatic backups and monitoring"
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
