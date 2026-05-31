---
title: "F2 — FSx Restore Issues"
description: "Diagnose restore operation failures from FSx backups"
status: active
severity: HIGH
triggers:
  - "restore failed"
  - "restore from backup"
  - "backup restore error"
  - "cannot restore"
  - "restore timeout"
owner: devops-agent
objective: "Identify and resolve issues when restoring FSx file systems from backups"
context: "FSx restore creates a new file system from a backup. Restore failures can be caused by subnet capacity, security group configuration, KMS key access, AD availability (Windows), or incompatible configuration parameters."
---

## Phase 1 — Triage

MUST:
- Describe the backup being restored: `aws fsx describe-backups --backup-ids <backup-id>`
- Check the new file system status: `aws fsx describe-file-systems --file-system-ids <new-fs-id>`
- Verify subnet has available IPs: `aws ec2 describe-subnets --subnet-ids <subnet-id>`
- Check KMS key accessibility: `aws kms describe-key --key-id <key-id>`

SHOULD:
- Verify AD is available for Windows FSx restores: `aws ds describe-directories`
- Check security group configuration for the restored file system
- Verify the restore parameters are compatible with the backup type
- Check CloudTrail for restore errors

MAY:
- Compare backup file system configuration with restore parameters
- Verify service quotas allow creating a new file system
- Check for region-specific limitations

## Phase 2 — Remediate

MUST:
- Fix the identified configuration issue (subnet, KMS, AD, security group)
- Ensure all restore parameters are compatible with the backup
- Retry the restore operation after fixing the issue

SHOULD:
- Use the same VPC/subnet configuration as the original file system when possible
- Verify AD connectivity before restoring Windows FSx
- Document the restore procedure for disaster recovery

MAY:
- Test restore procedures periodically as part of DR planning
- Automate restore testing with AWS Backup restore testing

## Common Issues

- symptoms: "Restore fails with subnet error"
  diagnosis: "Target subnet lacks available IP addresses or is in an unsupported AZ."
  resolution: "Choose a subnet with available IPs in a supported AZ."

- symptoms: "Windows FSx restore fails with AD error"
  diagnosis: "Active Directory is not reachable from the target subnet."
  resolution: "Verify AD connectivity, DNS resolution, and security group rules from the target subnet."

- symptoms: "Restore creates file system but it enters FAILED state"
  diagnosis: "KMS key is not accessible or has been deleted/disabled."
  resolution: "Ensure the KMS key is enabled and the FSx service role has access."

## Output Format

```yaml
root_cause: "restore_issue — <specific_cause>"
evidence:
  - type: backup
    content: "<backup details and configuration>"
  - type: file_system
    content: "<restored file system status and errors>"
  - type: infrastructure
    content: "<subnet, KMS, AD, security group status>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and retry restore"
  long_term: "Document and test restore procedures regularly"
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
