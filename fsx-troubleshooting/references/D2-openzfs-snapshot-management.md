---
title: "D2 — OpenZFS Snapshot Management Issues"
description: "Diagnose snapshot creation, retention, and space issues on FSx for OpenZFS"
status: active
severity: MEDIUM
triggers:
  - "openzfs snapshot"
  - "snapshot failed"
  - "snapshot space"
  - "snapshot restore"
  - "snapshot retention"
owner: devops-agent
objective: "Identify and resolve OpenZFS snapshot management issues including creation, retention, and space consumption"
context: "OpenZFS uses copy-on-write snapshots that are space-efficient but accumulate space as data changes. Snapshot issues typically involve space consumption, retention policy management, snapshot creation failures, or restore operations."
---

## Phase 1 — Triage

MUST:
- List snapshots for the volume: `aws fsx describe-snapshots --filters Name=volume-id,Values=<vol-id>`
- Check snapshot Lifecycle status
- Verify volume has available capacity: `aws fsx describe-volumes --volume-ids <vol-id>`
- Check file system storage capacity: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name FreeStorageCapacity --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`

SHOULD:
- Count total snapshots per volume (check against limits)
- Review snapshot creation timestamps for retention compliance
- Check if automatic snapshot policy is configured
- Verify snapshot is not in use by a clone volume

MAY:
- Calculate snapshot space consumption trends
- Review snapshot naming conventions for policy compliance
- Check for orphaned snapshots from deleted clone volumes

## Phase 2 — Remediate

MUST:
- Delete unnecessary snapshots to reclaim space
- Fix snapshot creation configuration if snapshots are failing
- Ensure sufficient storage capacity for snapshot overhead

SHOULD:
- Configure automatic snapshot policies with appropriate retention
- Monitor snapshot space consumption trends
- Set up alerts for storage capacity thresholds

MAY:
- Implement snapshot lifecycle automation using Lambda
- Use snapshots for creating clone volumes for test/dev

## Common Issues

- symptoms: "Snapshot creation fails with insufficient space"
  diagnosis: "File system storage is full — no space for new snapshot metadata."
  resolution: "Delete old snapshots or increase file system storage capacity."

- symptoms: "Storage usage keeps growing despite deleting files"
  diagnosis: "Snapshots retain references to deleted data blocks (copy-on-write)."
  resolution: "Delete snapshots that reference the old data. Space is reclaimed when all references are removed."

- symptoms: "Cannot delete a snapshot"
  diagnosis: "The snapshot is the parent of an active clone volume."
  resolution: "Delete or split the clone volume before deleting the parent snapshot."

## Output Format

```yaml
root_cause: "openzfs_snapshot — <specific_cause>"
evidence:
  - type: snapshots
    content: "<snapshot list with status and timestamps>"
  - type: volume
    content: "<volume capacity and usage>"
  - type: storage
    content: "<file system free storage capacity>"
severity: MEDIUM
mitigation:
  immediate: "Delete unnecessary snapshots or increase storage capacity"
  long_term: "Implement automated snapshot lifecycle management"
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
