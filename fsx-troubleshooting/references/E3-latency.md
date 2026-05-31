---
title: "E3 — Latency Issues"
description: "Diagnose high latency on FSx file system operations"
status: active
severity: HIGH
triggers:
  - "high latency"
  - "slow response"
  - "file operation slow"
  - "metadata latency"
  - "read latency"
  - "write latency"
owner: devops-agent
objective: "Identify and resolve latency issues affecting FSx file system performance"
context: "Latency on FSx can be caused by network distance (cross-AZ access), storage backend saturation, metadata-heavy operations, client configuration, or file system type characteristics. Each FSx type has different baseline latency expectations."
---

## Phase 1 — Triage

MUST:
- Describe the file system: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DataReadLatency --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check write latency: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DataWriteLatency --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify client is in the same AZ as the file system

SHOULD:
- Check IOPS and throughput utilization (saturation causes latency)
- Verify client mount options are optimized
- Check for metadata-heavy operations (many small files, directory listings)
- Compare latency across different operation types (read, write, metadata)

MAY:
- Test latency from different client instance types
- Check network latency between client and file system ENI
- Review application-level I/O patterns for optimization

## Phase 2 — Remediate

MUST:
- Place clients in the same AZ as the file system for lowest latency
- Resolve any IOPS or throughput saturation causing queuing delays
- Verify client mount options are appropriate for the workload

SHOULD:
- Use SSD storage for latency-sensitive workloads
- Optimize application I/O patterns (reduce metadata operations, batch I/O)
- Monitor latency trends for early detection of degradation

MAY:
- Use placement groups for client-to-FSx proximity
- Consider caching frequently accessed data on local instance storage

## Common Issues

- symptoms: "Latency spikes during specific time windows"
  diagnosis: "Backup operations or storage efficiency jobs are running during those windows."
  resolution: "Schedule backups and maintenance during low-activity periods."

- symptoms: "Cross-AZ clients experience higher latency"
  diagnosis: "Network round-trip time between AZs adds 1-2ms latency per operation."
  resolution: "Place latency-sensitive clients in the same AZ as the file system."

- symptoms: "Metadata operations (ls, stat) are slow on Lustre"
  diagnosis: "Large directories or many small files cause metadata server bottlenecks."
  resolution: "Use Lustre striping, reduce directory sizes, batch metadata operations."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<read/write/metadata latency measurements>"
  - type: saturation
    content: "<IOPS and throughput utilization>"
  - type: network
    content: "<client-to-FSx network path and AZ placement>"
severity: HIGH
mitigation:
  immediate: "Optimize client placement and resolve saturation"
  long_term: "Right-size file system and optimize workload I/O patterns"
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
