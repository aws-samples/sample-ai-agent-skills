---
title: "I2 — Access Control Issues"
description: "Diagnose file-level and API-level access control problems on FSx"
status: active
severity: MEDIUM
triggers:
  - "access denied"
  - "permission denied"
  - "IAM policy"
  - "file permissions"
  - "POSIX permissions"
  - "NTFS ACL"
owner: devops-agent
objective: "Identify and resolve access control issues at both the AWS API and file system level"
context: "FSx access control operates at two levels: AWS API level (IAM policies for FSx API calls) and file system level (POSIX for Lustre/OpenZFS, NTFS ACLs for Windows, UNIX/NTFS for ONTAP). Issues can stem from IAM policies, file permissions, NFS export policies, or AD group membership."
---

## Phase 1 — Triage

MUST:
- Identify the access control level (API vs file system)
- For API access: check IAM policies: `aws iam get-user-policy --user-name <user> --policy-name <policy>`
- For file access: check file system type and its permission model
- Verify security group allows the required protocol port

SHOULD:
- For Windows: check AD group membership and NTFS ACLs
- For Lustre/OpenZFS: check POSIX permissions and NFS export policies
- For ONTAP: check both UNIX permissions and NFS export policies or NTFS ACLs
- Check for SCP restrictions: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`

MAY:
- Use IAM Access Analyzer for API permission analysis
- Check for resource-based policies on the file system
- Review CloudTrail for access denied events

## Phase 2 — Remediate

MUST:
- Fix the identified permission issue (IAM policy, file permissions, export policy)
- Ensure the correct permission model is used for the file system type
- Verify the user/role has the minimum required permissions

SHOULD:
- Use IAM policies for API access control and file system native permissions for data access
- Follow least-privilege principle for both API and file access
- Document access control architecture

MAY:
- Implement centralized access management with AD groups (Windows/ONTAP)
- Set up access auditing for compliance

## Common Issues

- symptoms: "IAM user cannot describe or manage FSx file systems"
  diagnosis: "IAM policy lacks fsx:Describe* or fsx:* permissions."
  resolution: "Attach appropriate FSx IAM policy to the user/role."

- symptoms: "NFS client gets permission denied on Lustre/OpenZFS"
  diagnosis: "NFS export policy does not include the client IP or root_squash maps root to nobody."
  resolution: "Update NFS export to include client CIDR. Use no_root_squash if root access is needed."

- symptoms: "Windows user cannot access files despite share access"
  diagnosis: "NTFS ACLs on the files/folders deny the user's AD group."
  resolution: "Update NTFS ACLs to grant the user's AD group appropriate permissions."

## Output Format

```yaml
root_cause: "access_control — <specific_cause>"
evidence:
  - type: iam_policy
    content: "<IAM policy analysis>"
  - type: file_permissions
    content: "<file system level permissions>"
  - type: export_policy
    content: "<NFS export or share permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix the identified permission at the appropriate level"
  long_term: "Implement consistent access control model and documentation"
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
