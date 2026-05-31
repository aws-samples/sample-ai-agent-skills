---
title: "B3 — DFS Namespace Issues"
description: "Diagnose DFS namespace configuration problems with Windows FSx"
status: active
severity: MEDIUM
triggers:
  - "DFS namespace"
  - "DFS not working"
  - "DFS referral"
  - "namespace resolution"
  - "DFS folder target"
owner: devops-agent
objective: "Identify and resolve DFS namespace issues affecting Windows FSx share access"
context: "FSx for Windows supports DFS Namespaces for organizing shares across multiple file systems under a unified namespace. Issues typically involve AD configuration, DFS referral failures, namespace server registration, or folder target accessibility."
---

## Phase 1 — Triage

MUST:
- Describe the file system: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Verify the file system has DFS namespace capability (Windows FSx supports domain-based DFS)
- Check AD connectivity and domain controller health: `aws ds describe-directories`
- Verify security group allows DFS ports (445, 139): `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check DFS namespace configuration via PowerShell: `Get-DfsnRoot` and `Get-DfsnFolder`
- Verify DFS referral responses from the namespace server
- Check if the FSx file system is registered as a namespace server
- Verify DNS resolution for the DFS namespace path

MAY:
- Check DFS replication group configuration if using DFS-R
- Review Windows event logs for DFS-related errors
- Test DFS path access from multiple client locations

## Phase 2 — Remediate

MUST:
- Ensure the FSx file system is properly joined to the AD domain
- Verify DFS namespace server registration
- Confirm folder targets point to valid, accessible FSx shares

SHOULD:
- Use domain-based DFS namespaces for multi-site access
- Configure DFS referral ordering for optimal client routing
- Document the DFS namespace structure and folder targets

MAY:
- Set up DFS namespace monitoring
- Configure referral TTL for appropriate caching behavior

## Common Issues

- symptoms: "DFS namespace path not accessible"
  diagnosis: "The FSx file system is not registered as a DFS namespace server or AD connectivity is broken."
  resolution: "Re-register the FSx file system as a DFS namespace server. Verify AD connectivity."

- symptoms: "DFS referral returns wrong target"
  diagnosis: "Folder target configuration points to an incorrect or unavailable share."
  resolution: "Update DFS folder targets to point to the correct FSx share paths."

- symptoms: "DFS works for some users but not others"
  diagnosis: "DFS access-based enumeration is filtering folders based on user permissions."
  resolution: "Verify user permissions on the underlying share targets. Check ABE configuration."

## Output Format

```yaml
root_cause: "dfs_namespace — <specific_cause>"
evidence:
  - type: file_system
    content: "<FSx configuration and DFS capability>"
  - type: dfs_config
    content: "<DFS namespace and folder target configuration>"
  - type: ad_connectivity
    content: "<AD domain controller health and DNS>"
severity: MEDIUM
mitigation:
  immediate: "Fix DFS namespace registration or folder target configuration"
  long_term: "Document and standardize DFS namespace architecture"
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
