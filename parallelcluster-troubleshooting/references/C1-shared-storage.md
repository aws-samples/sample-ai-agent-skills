---
title: "C1 — Shared Storage Issues (EFS/FSx)"
description: "Diagnose shared storage mount and performance problems"
status: active
severity: HIGH
triggers:
  - "shared storage"
  - "EFS mount"
  - "FSx mount"
  - "storage performance"
  - "NFS"
owner: devops-agent
objective: "Resolve shared storage mount failures and performance issues"
context: "ParallelCluster supports EFS and FSx for Lustre as shared storage. EFS provides elastic NFS. FSx for Lustre provides high-performance parallel filesystem. Mount failures prevent compute nodes from becoming available."
---

## Phase 1 — Triage

MUST:
- Check EFS status: `aws efs describe-file-systems --query 'FileSystems[?Tags[?Key==`parallelcluster:cluster-name`]]'`
- Check FSx status: `aws fsx describe-file-systems --query 'FileSystems[*].{Id:FileSystemId,Type:FileSystemType,Status:Lifecycle}'`
- Verify mount targets (EFS): `aws efs describe-mount-targets --file-system-id <fs-id>`
- Check security groups allow NFS (port 2049) or Lustre (port 988)
- On head node: `df -h` and `mount | grep -E 'efs|fsx|nfs'`

SHOULD:
- Check EFS throughput mode: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[*].ThroughputMode'`
- Check FSx performance metrics: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name FreeDataStorageCapacity --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify mount points in cluster configuration

MAY:
- Check EFS burst credits if using bursting throughput
- Review FSx Lustre client logs on compute nodes

## Phase 2 — Remediate

MUST:
- If mount failure: verify security groups allow NFS/Lustre ports from cluster subnets
- If EFS mount target missing: create mount target in the cluster subnet
- If FSx not accessible: check VPC routing and security groups

SHOULD:
- Use provisioned throughput for EFS if burst credits are exhausted
- Monitor storage performance metrics
- Ensure mount targets exist in all AZs used by the cluster

MAY:
- Migrate from EFS to FSx for Lustre for HPC workloads needing higher throughput
- Implement S3 data repository association for FSx for Lustre

## Common Issues

- symptoms: "Compute nodes stuck in drain — mount failed"
  diagnosis: "Shared storage mount failure during node boot."
  resolution: "Check security groups for NFS/Lustre ports. Verify mount targets in subnet."

- symptoms: "Slow I/O on shared storage"
  diagnosis: "EFS burst credits exhausted or FSx throughput limit reached."
  resolution: "Switch to provisioned throughput (EFS) or scale FSx capacity."

## Output Format

```yaml
root_cause: "shared_storage — <specific_cause>"
evidence:
  - type: filesystem_status
    content: "<EFS/FSx status>"
  - type: mount_status
    content: "<mount point status>"
severity: HIGH
mitigation:
  immediate: "Fix mount configuration or security groups"
  long_term: "Implement storage monitoring and appropriate throughput mode"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — shared storage contains HPC job data and results
- Service Disruption Risk: HIGH — storage failures affect all cluster jobs
- Compliance Impact: HIGH — data storage must meet research data compliance requirements

## Escalation Conditions
- Shared storage becomes inaccessible to compute nodes
- Storage performance degradation affects job execution
- Storage capacity approaching limits
- Data corruption detected on shared storage
- Storage mount failures across multiple nodes

## Data Sensitivity
- HPC job data and results — CONFIDENTIAL
- Research datasets — CONFIDENTIAL
- Storage configurations — INTERNAL
- Mount point details — INTERNAL
- Never expose research data or job outputs

## Prohibited Actions
- Do NOT delete shared storage data without researcher approval
- Do NOT disable storage encryption
- Do NOT ignore storage capacity warnings
- Do NOT modify storage mount configurations during active jobs
- Do NOT share storage access with unauthorized users
- Do NOT bypass storage backup procedures

## Phase 3 — Rollback
- Restore shared storage from backups if data corruption detected
- Revert mount configuration changes
- Re-mount storage on affected compute nodes
- Restore storage capacity by cleaning temporary files
- Verify data integrity after storage recovery
- Document storage issues and update backup procedures

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
