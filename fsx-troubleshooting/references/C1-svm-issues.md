---
title: "C1 — ONTAP SVM Issues"
description: "Diagnose Storage Virtual Machine configuration and access problems on FSx for ONTAP"
status: active
severity: HIGH
triggers:
  - "SVM failed"
  - "storage virtual machine"
  - "SVM not accessible"
  - "SVM creation error"
  - "ONTAP SVM"
  - "virtual file server"
owner: devops-agent
objective: "Identify and resolve SVM configuration, creation, and access issues on FSx for ONTAP"
context: "SVMs are virtual file servers on ONTAP that provide multi-tenancy and protocol isolation. Each SVM has its own DNS name, network interfaces, and protocol configuration. SVM issues can stem from creation failures, network configuration, protocol settings, or Active Directory integration for SMB."
---

## Phase 1 — Triage

MUST:
- List SVMs on the file system: `aws fsx describe-storage-virtual-machines --filters Name=file-system-id,Values=<fs-id>`
- Check SVM Lifecycle status and FailureDetails
- Describe the parent file system: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Verify security group allows ONTAP ports (111, 635, 2049, 4046, 11104, 11105): `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check SVM Active Directory configuration if SMB is enabled
- Verify DNS resolution for the SVM DNS name
- List volumes in the SVM: `aws fsx describe-volumes --filters Name=storage-virtual-machine-id,Values=<svm-id>`
- Check CloudTrail for SVM-related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`

MAY:
- Verify NFS export policies on the SVM
- Check iSCSI LUN configuration if iSCSI is enabled
- Review SVM name mapping rules for cross-protocol access

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (network, AD, protocol configuration)
- Ensure security group allows all required ONTAP ports
- Verify SVM DNS name resolves to the correct network interface

SHOULD:
- Configure NFS and SMB protocols based on workload requirements
- Set up proper name mapping for cross-protocol access
- Document SVM configuration and protocol settings

MAY:
- Create separate SVMs for different workloads or teams
- Configure SVM-level snapshots and backup policies

## Common Issues

- symptoms: "SVM creation fails with AD join error"
  diagnosis: "SMB-enabled SVM cannot join the Active Directory domain."
  resolution: "Verify AD credentials, DNS resolution, and security group allows AD ports (53, 88, 389, 445)."

- symptoms: "NFS mount fails to SVM endpoint"
  diagnosis: "Security group blocks NFS port 2049 or export policy denies the client."
  resolution: "Add inbound TCP/UDP 2049 to security group. Verify NFS export policy allows the client IP."

- symptoms: "Cross-protocol access denied (NFS user accessing SMB data)"
  diagnosis: "Name mapping rules are not configured to map UNIX users to Windows users."
  resolution: "Configure name mapping rules on the SVM to map between UNIX and Windows identities."

## Output Format

```yaml
root_cause: "svm_issue — <specific_cause>"
evidence:
  - type: svm_status
    content: "<describe-storage-virtual-machines output>"
  - type: file_system
    content: "<parent file system configuration>"
  - type: network
    content: "<security group rules and DNS resolution>"
severity: HIGH
mitigation:
  immediate: "Fix SVM configuration or network connectivity"
  long_term: "Standardize SVM deployment with IaC templates"
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
