---
title: "C2 — Scratch Storage Issues"
description: "Diagnose scratch filesystem problems and data persistence concerns"
status: active
severity: MEDIUM
triggers:
  - "scratch storage"
  - "scratch filesystem"
  - "temporary storage"
  - "FSx scratch"
owner: devops-agent
objective: "Resolve scratch storage configuration and data management issues"
context: "Scratch storage in ParallelCluster typically uses FSx for Lustre SCRATCH deployment types or local instance storage. Scratch data is NOT persistent — it is lost when the cluster or filesystem is deleted."
---

## Phase 1 — Triage

MUST:
- Check FSx filesystem type: `aws fsx describe-file-systems --file-system-id <fs-id> --query 'FileSystems[*].{Id:FileSystemId,DeploymentType:LustreConfiguration.DeploymentType}'`
- Verify mount on head node: `df -h /scratch` (or configured mount point)
- Check filesystem capacity: `aws fsx describe-file-systems --file-system-id <fs-id> --query 'FileSystems[*].StorageCapacity'`
- Verify cluster configuration for scratch storage settings

SHOULD:
- Check S3 data repository association if configured
- Monitor filesystem utilization
- Verify scratch is accessible from compute nodes

MAY:
- Check FSx Lustre performance metrics
- Review data import/export status from S3

## Phase 2 — Remediate

MUST:
- If scratch full: increase FSx capacity or clean up old data
- If mount failure: check security groups and VPC configuration
- If data needed after deletion: back up to S3 BEFORE deleting cluster

SHOULD:
- Configure S3 data repository for automatic data persistence
- Use PERSISTENT deployment type if data must survive cluster deletion
- Implement data lifecycle management

MAY:
- Set up automated backup scripts to S3
- Use FSx data repository tasks for bulk data movement

## Common Issues

- symptoms: "Data lost after cluster deletion"
  diagnosis: "FSx SCRATCH deployment type — data is not persistent."
  resolution: "Use PERSISTENT type or back up to S3 before deletion."

- symptoms: "Scratch filesystem full"
  diagnosis: "Job output or temporary files filling the filesystem."
  resolution: "Clean up old data. Increase FSx capacity if needed."

## Output Format

```yaml
root_cause: "scratch — <specific_cause>"
evidence:
  - type: filesystem_config
    content: "<FSx deployment type and capacity>"
  - type: utilization
    content: "<storage utilization>"
severity: MEDIUM
mitigation:
  immediate: "Free space or fix mount issue"
  long_term: "Implement S3 data repository and lifecycle management"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — scratch storage may contain intermediate job data
- Service Disruption Risk: MEDIUM — scratch issues affect job performance
- Compliance Impact: LOW — scratch storage is temporary by nature

## Escalation Conditions
- Scratch storage becomes unavailable
- Scratch performance degradation affects job completion times
- Scratch capacity exhausted preventing job execution
- FSx for Lustre issues affecting scratch performance
- Scratch data persistence issues after node termination

## Data Sensitivity
- Intermediate job data — INTERNAL (may contain CONFIDENTIAL data)
- Scratch configuration — INTERNAL
- FSx for Lustre settings — INTERNAL
- Performance metrics — INTERNAL
- Never assume scratch data is automatically backed up

## Prohibited Actions
- Do NOT store critical data only on scratch storage
- Do NOT ignore scratch capacity warnings
- Do NOT modify scratch configurations during active jobs
- Do NOT disable scratch storage encryption for sensitive workloads
- Do NOT share scratch storage across security boundaries
- Do NOT rely on scratch data persistence after cluster operations

## Phase 3 — Rollback
- Restore scratch storage configuration to previous settings
- Re-create FSx for Lustre filesystem if corrupted
- Revert scratch mount configurations
- Verify scratch accessibility from compute nodes
- Re-submit jobs that failed due to scratch issues
- Document scratch storage issues and capacity requirements

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
