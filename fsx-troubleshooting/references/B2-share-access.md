---
title: "B2 — Windows Share Access Issues"
description: "Diagnose SMB share access problems on FSx for Windows File Server"
status: active
severity: MEDIUM
triggers:
  - "cannot access share"
  - "share access denied"
  - "SMB connection failed"
  - "windows mount failed"
  - "permission denied windows fsx"
owner: devops-agent
objective: "Identify and resolve SMB share access issues on Windows FSx file systems"
context: "Windows FSx share access requires proper AD authentication, security group rules allowing port 445, correct NTFS and share-level permissions, and network connectivity. Issues can stem from AD group membership, share permissions, NTFS ACLs, or DNS resolution."
---

## Phase 1 — Triage

MUST:
- Describe the file system for DNS name and status: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Verify security group allows SMB port 445: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check the file system Lifecycle is AVAILABLE
- Verify the client is domain-joined and can resolve the FSx DNS name

SHOULD:
- Check if the user/group has share-level permissions (via fsmgmt.msc or PowerShell)
- Verify NTFS permissions on the shared folder
- Test DNS resolution of the FSx DNS name from the client
- Check for GPO restrictions that may block SMB connections

MAY:
- Check Windows event logs on the client for SMB errors
- Verify SMB protocol version compatibility (FSx supports SMB 2.0-3.1.1)
- Test connectivity with `Test-NetConnection <fsx-dns-name> -Port 445` from the client

## Phase 2 — Remediate

MUST:
- Ensure security group allows inbound TCP 445 from client subnets
- Verify the user is a member of the appropriate AD group with share access
- Confirm the file system DNS name resolves correctly from the client

SHOULD:
- Use AD groups for share permissions rather than individual users
- Set both share-level and NTFS permissions appropriately
- Document share permission structure

MAY:
- Configure DFS namespaces for simplified share access
- Set up auditing for share access failures

## Common Issues

- symptoms: "Access denied when mapping network drive"
  diagnosis: "User lacks share-level or NTFS permissions on the FSx file share."
  resolution: "Grant the user's AD group appropriate share and NTFS permissions via fsmgmt.msc."

- symptoms: "Cannot connect to \\\\fsx-dns-name\\share"
  diagnosis: "DNS resolution fails or security group blocks port 445."
  resolution: "Verify DNS resolution. Add inbound TCP 445 rule to the FSx security group."

- symptoms: "Connection works from some clients but not others"
  diagnosis: "Security group restricts access to specific subnets or CIDR ranges."
  resolution: "Update security group to allow TCP 445 from all client subnets."

## Output Format

```yaml
root_cause: "share_access — <specific_cause>"
evidence:
  - type: file_system
    content: "<DNS name, status, security group>"
  - type: permissions
    content: "<share-level and NTFS permission analysis>"
  - type: network
    content: "<connectivity and DNS resolution test results>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions or network connectivity"
  long_term: "Standardize share permission model using AD groups"
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
