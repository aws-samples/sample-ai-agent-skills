---
title: "C2 — ONTAP Volume Management Issues"
description: "Diagnose volume creation, sizing, and tiering problems on FSx for ONTAP"
status: active
severity: MEDIUM
triggers:
  - "ONTAP volume"
  - "volume creation failed"
  - "volume full"
  - "tiering policy"
  - "volume resize"
  - "junction path"
owner: devops-agent
objective: "Identify and resolve ONTAP volume management issues including creation, sizing, tiering, and access"
context: "ONTAP volumes are created within SVMs and support thin provisioning, tiering to capacity pool storage, snapshots, and FlexClone. Volume issues can involve capacity management, tiering policy configuration, junction path mounting, or snapshot space consumption."
---

## Phase 1 — Triage

MUST:
- List volumes on the file system: `aws fsx describe-volumes --filters Name=file-system-id,Values=<fs-id>`
- Check volume Lifecycle status and size configuration
- Verify the parent SVM is healthy: `aws fsx describe-storage-virtual-machines --filters Name=file-system-id,Values=<fs-id>`
- Check file system storage capacity: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name StorageCapacity --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Check volume tiering policy configuration (AUTO, SNAPSHOT_ONLY, ALL, NONE)
- Verify junction path is set correctly for NFS/SMB access
- Check snapshot space consumption on the volume
- Monitor StorageUsed metric for capacity trends

MAY:
- Review FlexClone volumes and their parent relationships
- Check volume efficiency settings (deduplication, compression)
- Verify volume security style (UNIX, NTFS, MIXED) matches access protocol

## Phase 2 — Remediate

MUST:
- Fix the identified volume configuration issue
- Ensure volume size does not exceed SVM or file system capacity limits
- Verify junction path is accessible from clients

SHOULD:
- Configure appropriate tiering policy for the workload (AUTO for general, SNAPSHOT_ONLY for active data)
- Set up volume autosize policies to handle growth
- Schedule snapshot policies to manage snapshot space

MAY:
- Enable volume compression and deduplication for storage efficiency
- Use FlexClone for instant test/dev copies

## Common Issues

- symptoms: "Volume creation fails with insufficient capacity"
  diagnosis: "The file system SSD storage is fully allocated across existing volumes."
  resolution: "Increase file system SSD storage capacity or reduce existing volume sizes."

- symptoms: "Volume appears full but data usage is low"
  diagnosis: "Snapshot reserve or snapshot copies are consuming volume space."
  resolution: "Delete old snapshots or increase snapshot reserve. Check snapshot policy schedule."

- symptoms: "Data not tiering to capacity pool storage"
  diagnosis: "Tiering policy is set to NONE or cooling period has not elapsed."
  resolution: "Set tiering policy to AUTO or ALL. Default cooling period is 31 days for AUTO."

## Output Format

```yaml
root_cause: "volume_management — <specific_cause>"
evidence:
  - type: volume_config
    content: "<describe-volumes output with size and tiering>"
  - type: capacity
    content: "<file system and SVM capacity analysis>"
  - type: snapshots
    content: "<snapshot space consumption>"
severity: MEDIUM
mitigation:
  immediate: "Fix volume configuration or capacity issue"
  long_term: "Implement volume autosize and snapshot lifecycle policies"
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
