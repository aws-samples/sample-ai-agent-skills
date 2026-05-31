---
title: "G1 — Storage Capacity Scaling Issues"
description: "Diagnose storage capacity increase failures and planning issues"
status: active
severity: MEDIUM
triggers:
  - "storage capacity"
  - "increase storage"
  - "scale storage"
  - "disk full"
  - "storage scaling failed"
  - "capacity increase"
owner: devops-agent
objective: "Identify and resolve storage capacity scaling issues on FSx file systems"
context: "Storage capacity can only be increased (never decreased) on all FSx types. Lustre requires capacity increases in specific increments. Scaling takes time and the file system remains available during the operation. Scaling failures can be caused by invalid increment sizes, service quotas, or ongoing operations."
---

## Phase 1 — Triage

MUST:
- Describe the file system for current capacity: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check if a scaling operation is in progress (AdministrativeActions)
- Verify free storage capacity: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name FreeStorageCapacity --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`
- Check service quotas: `aws service-quotas get-service-quota --service-code fsx --quota-code L-A67D2BBF`

SHOULD:
- Verify the requested capacity increase meets minimum increment requirements
- Check for any ongoing administrative actions that may block scaling
- Review storage capacity trends to plan appropriate increase
- Check CloudTrail for scaling errors

MAY:
- Calculate storage growth rate for capacity planning
- Review ONTAP volume allocations if scaling ONTAP SSD storage
- Check Lustre OST distribution after capacity increase

## Phase 2 — Remediate

MUST:
- Ensure the capacity increase meets the minimum increment for the file system type
- Wait for any ongoing administrative actions to complete before scaling
- Verify the new capacity does not exceed service quotas

SHOULD:
- Plan capacity increases proactively based on growth trends
- Set up CloudWatch alarms for low free storage capacity
- For Lustre, understand that capacity increases add new OSTs (not expand existing ones)

MAY:
- Automate capacity scaling with CloudWatch alarms and Lambda
- Document capacity planning procedures

## Common Issues

- symptoms: "Storage scaling request rejected with invalid increment"
  diagnosis: "Lustre requires capacity increases in multiples of the original storage capacity chunk size."
  resolution: "For Lustre scratch: multiples of 2.4 TiB. For persistent SSD: multiples of 1.2 TiB. For persistent HDD: multiples of 6 TiB."

- symptoms: "Scaling operation stuck in PENDING"
  diagnosis: "Another administrative action is in progress on the file system."
  resolution: "Wait for the current operation to complete. Only one administrative action can run at a time."

- symptoms: "Want to decrease storage capacity"
  diagnosis: "Storage capacity cannot be decreased on any FSx file system type."
  resolution: "Create a new smaller file system and migrate data. Storage scaling is one-way (increase only)."

## Output Format

```yaml
root_cause: "storage_scaling — <specific_cause>"
evidence:
  - type: file_system
    content: "<current capacity and administrative actions>"
  - type: storage_metrics
    content: "<free storage capacity and trends>"
  - type: quotas
    content: "<service quota limits>"
severity: MEDIUM
mitigation:
  immediate: "Fix increment size or wait for pending operations"
  long_term: "Implement proactive capacity monitoring and scaling automation"
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
