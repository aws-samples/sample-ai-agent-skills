---
title: "G2 — Throughput Scaling Issues"
description: "Diagnose throughput capacity scaling problems on FSx file systems"
status: active
severity: MEDIUM
triggers:
  - "throughput scaling"
  - "increase throughput"
  - "change throughput"
  - "throughput capacity update"
  - "scale throughput"
owner: devops-agent
objective: "Identify and resolve throughput capacity scaling issues on FSx file systems"
context: "Throughput capacity can be independently scaled on Windows, ONTAP, and OpenZFS file systems. Lustre throughput is tied to storage capacity and per-unit-storage-throughput (immutable). Throughput changes may cause brief I/O pauses during the update."
---

## Phase 1 — Triage

MUST:
- Describe the file system for current throughput: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check for ongoing administrative actions (throughput update in progress)
- Identify the file system type (Lustre throughput cannot be independently scaled)
- Check throughput utilization: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name NetworkThroughputUtilization --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Verify the requested throughput value is valid for the file system type
- Check if the file system supports throughput scaling (Lustre per-unit-storage-throughput is immutable)
- Review throughput utilization trends to determine appropriate target
- Check for maintenance window requirements

MAY:
- Compare throughput costs at different capacity levels
- Review workload patterns to determine if scaling is needed vs optimization
- Check if burst credits can cover temporary throughput needs

## Phase 2 — Remediate

MUST:
- For Windows/ONTAP/OpenZFS: use update-file-system to change throughput capacity
- For Lustre: increase storage capacity (throughput scales with storage) or create new file system with higher per-unit-storage-throughput
- Plan for brief I/O pause during throughput update

SHOULD:
- Schedule throughput changes during maintenance windows
- Monitor performance after throughput change to verify improvement
- Right-size throughput based on sustained workload analysis

MAY:
- Automate throughput scaling based on CloudWatch metrics
- Use scheduled scaling for predictable workload patterns

## Common Issues

- symptoms: "Cannot change throughput on Lustre file system"
  diagnosis: "Lustre per-unit-storage-throughput is set at creation and cannot be changed."
  resolution: "Create a new Lustre file system with the desired per-unit-storage-throughput. Migrate data."

- symptoms: "Throughput update fails or takes very long"
  diagnosis: "Another administrative action is in progress or the file system is in a transitional state."
  resolution: "Wait for pending operations to complete. Verify file system is in AVAILABLE state."

- symptoms: "Brief I/O interruption during throughput change"
  diagnosis: "This is expected behavior — throughput changes cause a brief failover on Windows multi-AZ."
  resolution: "Schedule throughput changes during maintenance windows. Applications should handle brief disconnects."

## Output Format

```yaml
root_cause: "throughput_scaling — <specific_cause>"
evidence:
  - type: file_system
    content: "<current throughput configuration and administrative actions>"
  - type: throughput_metrics
    content: "<throughput utilization trends>"
  - type: file_system_type
    content: "<type-specific throughput scaling behavior>"
severity: MEDIUM
mitigation:
  immediate: "Update throughput capacity or create new file system for Lustre"
  long_term: "Implement throughput monitoring and right-sizing reviews"
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
