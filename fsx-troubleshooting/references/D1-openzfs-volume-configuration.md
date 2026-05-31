---
title: "D1 — OpenZFS Volume Configuration Issues"
description: "Diagnose volume configuration problems on FSx for OpenZFS"
status: active
severity: MEDIUM
triggers:
  - "openzfs volume"
  - "openzfs configuration"
  - "record size"
  - "openzfs NFS"
  - "openzfs mount"
  - "openzfs quota"
owner: devops-agent
objective: "Identify and resolve OpenZFS volume configuration issues including NFS exports, quotas, and record sizes"
context: "OpenZFS volumes support configurable record sizes, data compression, NFS export options, user/group quotas, and copy-on-write semantics. Issues typically involve NFS export configuration, record size mismatch for workload, quota enforcement, or compression settings."
---

## Phase 1 — Triage

MUST:
- List volumes: `aws fsx describe-volumes --filters Name=file-system-id,Values=<fs-id>`
- Check volume configuration (record size, compression, NFS exports, quotas)
- Verify the file system is healthy: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check security group allows NFS ports (111, 2049): `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Verify NFS export configuration (client CIDR, options like root_squash)
- Check volume data compression type (NONE, ZSTD, LZ4, ZSTD-N)
- Verify record size is appropriate for the workload (default 128K)
- Check user/group quota configuration and enforcement

MAY:
- Monitor volume storage usage via CloudWatch
- Check parent-child volume hierarchy
- Verify copy-on-write snapshot space consumption

## Phase 2 — Remediate

MUST:
- Fix NFS export configuration to allow client access
- Ensure security group allows NFS traffic from client subnets
- Verify volume path is correct for NFS mount

SHOULD:
- Set record size to match workload I/O pattern (4K for databases, 128K for general, 1M for large sequential)
- Enable compression (LZ4 for balanced, ZSTD for maximum compression)
- Configure appropriate quotas to prevent runaway storage consumption

MAY:
- Create child volumes for workload isolation
- Set up volume-level snapshot schedules

## Common Issues

- symptoms: "NFS mount fails with permission denied"
  diagnosis: "NFS export configuration does not include the client IP/CIDR or uses restrictive options."
  resolution: "Update volume NFS exports to include the client CIDR with appropriate options."

- symptoms: "Poor performance for database workload on OpenZFS"
  diagnosis: "Record size is set to default 128K but database uses small random I/O."
  resolution: "Create a new volume with record size matching database block size (typically 4K-16K)."

- symptoms: "Volume shows more space used than expected"
  diagnosis: "Copy-on-write snapshots retain old data blocks, consuming additional space."
  resolution: "Delete unnecessary snapshots. Space is reclaimed when snapshot references are removed."

## Output Format

```yaml
root_cause: "openzfs_volume — <specific_cause>"
evidence:
  - type: volume_config
    content: "<volume configuration including NFS exports, record size, compression>"
  - type: file_system
    content: "<parent file system status>"
  - type: network
    content: "<security group and NFS connectivity>"
severity: MEDIUM
mitigation:
  immediate: "Fix NFS export or volume configuration"
  long_term: "Standardize volume templates with workload-appropriate settings"
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
