---
title: "E2 — IOPS Performance Issues"
description: "Diagnose IOPS bottlenecks and disk I/O performance problems on FSx"
status: active
severity: HIGH
triggers:
  - "IOPS limit"
  - "disk IOPS"
  - "I/O bottleneck"
  - "slow random I/O"
  - "IOPS throttling"
owner: devops-agent
objective: "Identify and resolve IOPS performance issues on FSx file systems"
context: "IOPS performance depends on file system type, storage type (SSD vs HDD), and provisioned IOPS configuration. ONTAP and OpenZFS support provisioned SSD IOPS. Lustre IOPS scales with storage capacity. Windows FSx IOPS depends on storage type and capacity."
---

## Phase 1 — Triage

MUST:
- Describe the file system for IOPS configuration: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check IOPS utilization: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DiskIopsUtilization --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check DataReadOperations and DataWriteOperations metrics
- Identify storage type (SSD vs HDD) and provisioned IOPS setting

SHOULD:
- Compare actual IOPS against provisioned/expected limits
- Check for IOPS burst credit exhaustion
- Analyze read vs write IOPS ratio
- Verify workload I/O size (small random I/O consumes more IOPS)

MAY:
- Check client-side I/O wait times
- Review application I/O patterns for optimization
- Compare IOPS across multiple volumes (ONTAP)

## Phase 2 — Remediate

MUST:
- Increase provisioned IOPS if consistently hitting limits (ONTAP/OpenZFS)
- For HDD-based systems, consider migrating to SSD for IOPS-sensitive workloads
- Verify the workload IOPS requirement matches the file system configuration

SHOULD:
- Use SSD storage for random I/O workloads
- Optimize application I/O patterns (batch small writes, use appropriate block sizes)
- Monitor IOPS utilization trends for capacity planning

MAY:
- Distribute I/O across multiple volumes for ONTAP
- Use caching layers for read-heavy IOPS workloads

## Common Issues

- symptoms: "DiskIopsUtilization consistently at 100%"
  diagnosis: "Workload IOPS demand exceeds provisioned IOPS capacity."
  resolution: "Increase provisioned SSD IOPS (ONTAP/OpenZFS) or increase SSD storage capacity."

- symptoms: "High IOPS on HDD-based Windows FSx"
  diagnosis: "HDD storage provides limited IOPS (12 per GiB). Random I/O workloads need SSD."
  resolution: "Migrate to SSD storage type or increase HDD capacity for more baseline IOPS."

- symptoms: "IOPS drops periodically then recovers"
  diagnosis: "IOPS burst credits are being exhausted and replenished cyclically."
  resolution: "Increase baseline IOPS capacity to match sustained workload requirements."

## Output Format

```yaml
root_cause: "iops_issue — <specific_cause>"
evidence:
  - type: iops_config
    content: "<provisioned IOPS, storage type, capacity>"
  - type: cloudwatch_metrics
    content: "<DiskIopsUtilization, read/write operations>"
  - type: workload
    content: "<I/O pattern analysis>"
severity: HIGH
mitigation:
  immediate: "Increase provisioned IOPS or optimize I/O patterns"
  long_term: "Right-size IOPS configuration based on workload analysis"
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
