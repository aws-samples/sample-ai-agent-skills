---
title: "A3 — Lustre Throughput Issues"
description: "Diagnose throughput performance problems on FSx for Lustre"
status: active
severity: MEDIUM
triggers:
  - "lustre slow"
  - "lustre throughput"
  - "lustre performance"
  - "lustre bandwidth"
  - "slow file operations lustre"
owner: devops-agent
objective: "Identify and resolve throughput bottlenecks on FSx for Lustre file systems"
context: "Lustre throughput is determined by storage capacity and deployment type. Scratch provides higher burst throughput. Persistent throughput depends on the per-unit-storage-throughput setting (50/100/200 MB/s per TiB). Throughput issues may also stem from client-side configuration, network bottlenecks, or workload patterns."
---

## Phase 1 — Triage

MUST:
- Describe the file system for throughput configuration: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check DataReadBytes and DataWriteBytes metrics: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DataReadBytes --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check storage capacity and deployment type (scratch vs persistent, per-unit-storage-throughput)
- Verify the file system is not in UPDATING or MISCONFIGURED state

SHOULD:
- Check FreeStorageCapacity to rule out full disk: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name FreeStorageCapacity --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Monitor MetadataOperations metric for metadata-heavy workloads
- Verify client mount options (flock, default stripe settings)
- Check if data is being lazy-loaded from S3 (first-access penalty)

MAY:
- Run lfs df on the Lustre client to check OST utilization
- Check client-side network throughput to the file system ENI
- Review workload I/O patterns (sequential vs random, file sizes)

## Phase 2 — Remediate

MUST:
- Verify throughput expectations match the deployment type and storage capacity
- For persistent Lustre, confirm per-unit-storage-throughput setting matches requirements

SHOULD:
- Pre-load data from S3 using import tasks to avoid first-access latency
- Optimize client mount options for the workload type
- Consider increasing storage capacity to increase aggregate throughput (Lustre scales with capacity)

MAY:
- Use Lustre file striping to distribute I/O across OSTs
- Consider scratch deployment for maximum burst throughput on temporary workloads

## Common Issues

- symptoms: "Throughput much lower than expected for storage capacity"
  diagnosis: "Per-unit-storage-throughput is set to 50 MB/s/TiB instead of 200 MB/s/TiB."
  resolution: "This setting is immutable after creation. Create a new file system with higher per-unit-storage-throughput."

- symptoms: "First access to files is very slow"
  diagnosis: "Data is being lazy-loaded from S3 via the data repository association."
  resolution: "Run an import data repository task to pre-load data before the workload starts."

- symptoms: "Throughput drops during metadata-heavy operations"
  diagnosis: "Small file operations (ls, stat) consume metadata IOPS, not data throughput."
  resolution: "Use Lustre file striping and batch metadata operations. Consider persistent SSD for metadata-heavy workloads."

## Output Format

```yaml
root_cause: "lustre_throughput — <specific_cause>"
evidence:
  - type: file_system_config
    content: "<deployment type, storage capacity, per-unit-storage-throughput>"
  - type: cloudwatch_metrics
    content: "<DataReadBytes, DataWriteBytes, FreeStorageCapacity>"
  - type: workload_pattern
    content: "<I/O pattern analysis>"
severity: MEDIUM
mitigation:
  immediate: "Pre-load data from S3 and optimize client mount options"
  long_term: "Right-size storage capacity and per-unit-storage-throughput for workload"
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
