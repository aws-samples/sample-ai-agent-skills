---
title: "E1 — Throughput Limit Issues"
description: "Diagnose throughput bottlenecks across all FSx file system types"
status: active
severity: HIGH
triggers:
  - "throughput limit"
  - "throughput bottleneck"
  - "slow file transfer"
  - "bandwidth limit"
  - "throughput capacity"
owner: devops-agent
objective: "Identify and resolve throughput limitations on FSx file systems"
context: "Throughput behavior differs by FSx type. Lustre scales with storage capacity. Windows, ONTAP, and OpenZFS have independently configurable throughput capacity. Throughput limits can be hit due to undersized configuration, burst credit exhaustion, or network bottlenecks."
---

## Phase 1 — Triage

MUST:
- Describe the file system for throughput configuration: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check throughput utilization: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name NetworkThroughputUtilization --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check DataReadBytes and DataWriteBytes: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name DataReadBytes --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Identify the file system type and its throughput model

SHOULD:
- Check burst credit balance (if applicable): `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name BurstCreditBalance --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Compare actual throughput against configured/expected limits
- Check client-side network throughput (EC2 instance network bandwidth)
- Verify no other workloads are competing for file system throughput

MAY:
- Analyze I/O patterns (sequential vs random, read vs write ratio)
- Check for metadata-heavy operations consuming throughput
- Review client mount options for optimization opportunities

## Phase 2 — Remediate

MUST:
- For Windows/ONTAP/OpenZFS: increase throughput capacity if consistently hitting limits
- For Lustre: increase storage capacity to increase aggregate throughput
- Verify client EC2 instance type supports required network bandwidth

SHOULD:
- Right-size throughput capacity based on sustained workload requirements
- Monitor burst credit balance to avoid exhaustion
- Distribute workload across multiple clients for parallel throughput

MAY:
- Use placement groups for lowest-latency client-to-FSx connectivity
- Optimize application I/O patterns for better throughput utilization

## Common Issues

- symptoms: "Throughput drops to baseline after initial burst"
  diagnosis: "Burst credits are exhausted. Sustained throughput falls to baseline level."
  resolution: "Increase throughput capacity to match sustained requirements. Monitor BurstCreditBalance."

- symptoms: "Throughput lower than configured capacity"
  diagnosis: "Client EC2 instance network bandwidth is the bottleneck, not the file system."
  resolution: "Use an EC2 instance type with higher network bandwidth. Check instance network performance."

- symptoms: "Windows FSx throughput unchanged after storage increase"
  diagnosis: "Windows FSx throughput is independent of storage capacity."
  resolution: "Increase throughput capacity separately using update-file-system."

## Output Format

```yaml
root_cause: "throughput_limit — <specific_cause>"
evidence:
  - type: throughput_config
    content: "<file system throughput configuration>"
  - type: cloudwatch_metrics
    content: "<throughput utilization, burst credits, data bytes>"
  - type: client_network
    content: "<client EC2 instance network bandwidth>"
severity: HIGH
mitigation:
  immediate: "Increase throughput capacity or optimize client configuration"
  long_term: "Right-size throughput based on sustained workload analysis"
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
