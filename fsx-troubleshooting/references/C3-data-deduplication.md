---
title: "C3 — ONTAP Data Deduplication Issues"
description: "Diagnose data deduplication and storage efficiency problems on FSx for ONTAP"
status: active
severity: LOW
triggers:
  - "deduplication"
  - "storage efficiency"
  - "compression"
  - "compaction"
  - "space savings"
owner: devops-agent
objective: "Identify and resolve data deduplication and compression issues on ONTAP volumes"
context: "ONTAP supports inline and background deduplication, compression (LZ4, ZSTD), and compaction. These features reduce storage consumption but can impact write performance. Issues typically involve efficiency not running, unexpected space usage, or performance impact from efficiency operations."
---

## Phase 1 — Triage

MUST:
- Describe volumes to check efficiency settings: `aws fsx describe-volumes --filters Name=file-system-id,Values=<fs-id>`
- Check volume OntapConfiguration for StorageEfficiencyEnabled flag
- Verify the file system is ONTAP type: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check storage capacity metrics: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name StorageUsed --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 3600 --statistics Average`

SHOULD:
- Check if deduplication is running or scheduled via ONTAP CLI (if accessible)
- Verify compression type configured on the volume
- Monitor write latency during efficiency operations
- Compare logical vs physical data usage for savings ratio

MAY:
- Review efficiency operation schedules
- Check if inline deduplication is enabled vs background only
- Analyze workload data patterns for deduplication potential

## Phase 2 — Remediate

MUST:
- Enable StorageEfficiencyEnabled on volumes where deduplication is desired
- Verify the volume has sufficient free space for deduplication metadata

SHOULD:
- Use inline deduplication for real-time savings on write
- Schedule background deduplication during low-activity periods
- Monitor write latency impact and adjust if needed

MAY:
- Enable compression alongside deduplication for maximum savings
- Use volume-level reporting to track efficiency ratios over time

## Common Issues

- symptoms: "No space savings despite enabling storage efficiency"
  diagnosis: "Data has low deduplication potential (unique data) or background dedup has not run yet."
  resolution: "Verify data patterns. Allow background dedup to complete. Check efficiency operation status."

- symptoms: "Write performance degraded after enabling efficiency"
  diagnosis: "Inline deduplication and compression add CPU overhead to write operations."
  resolution: "Consider using background-only deduplication or reducing compression level."

- symptoms: "Storage efficiency enabled but volume still shows high usage"
  diagnosis: "Snapshot copies retain pre-dedup data blocks, preventing space reclamation."
  resolution: "Delete old snapshots to allow dedup savings to be realized. New snapshots will reflect deduped data."

## Output Format

```yaml
root_cause: "data_deduplication — <specific_cause>"
evidence:
  - type: volume_efficiency
    content: "<volume efficiency configuration and status>"
  - type: storage_metrics
    content: "<logical vs physical usage, savings ratio>"
  - type: performance
    content: "<write latency during efficiency operations>"
severity: LOW
mitigation:
  immediate: "Verify efficiency settings and allow background operations to complete"
  long_term: "Tune efficiency settings based on workload patterns and performance requirements"
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
