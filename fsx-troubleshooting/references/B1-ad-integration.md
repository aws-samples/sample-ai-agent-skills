---
title: "B1 — Active Directory Integration Issues"
description: "Diagnose Windows FSx Active Directory join and integration failures"
status: active
severity: HIGH
triggers:
  - "AD join failed"
  - "active directory error"
  - "domain join failure"
  - "windows fsx creation failed"
  - "directory service"
  - "DNS resolution failed"
owner: devops-agent
objective: "Identify and resolve Active Directory integration issues preventing Windows FSx operation"
context: "FSx for Windows File Server requires Active Directory for authentication. Creation failures are commonly caused by DNS resolution issues, domain controller connectivity, incorrect AD credentials, OU permissions, or security group rules blocking AD ports (53, 88, 389, 445, 636, 3268, 9389)."
---

## Phase 1 — Triage

MUST:
- Describe the file system for AD configuration: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check FailureDetails for AD-specific error messages
- Verify the directory exists: `aws ds describe-directories`
- Check security group allows AD ports: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Verify DNS resolution from FSx subnets to domain controllers
- Check the AD service account has permissions to join computers to the specified OU
- Verify VPC DHCP options set includes AD DNS servers: `aws ec2 describe-dhcp-options --dhcp-options-ids <dhcp-id>`
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`

MAY:
- Verify AD trust relationships if using a resource forest
- Check AD replication health between domain controllers
- Test DNS resolution manually from an EC2 instance in the same subnet

## Phase 2 — Remediate

MUST:
- Ensure security group allows outbound to AD ports: TCP/UDP 53 (DNS), TCP/UDP 88 (Kerberos), TCP 389 (LDAP), TCP 445 (SMB), TCP 636 (LDAPS), TCP 3268 (Global Catalog)
- Verify VPC DNS settings point to AD domain controllers
- Ensure the AD service account has CreateComputer permission on the target OU

SHOULD:
- Use AWS Managed Microsoft AD for simplified integration
- Place FSx file system in the same VPC as the directory or ensure VPC peering/transit gateway connectivity
- Document the OU path and service account used for FSx

MAY:
- Set up monitoring for AD connectivity from FSx subnets
- Create a dedicated OU for FSx computer objects

## Common Issues

- symptoms: "File system FAILED with 'Unable to resolve DNS name for domain controller'"
  diagnosis: "VPC DHCP options do not include AD DNS servers or DNS is not reachable."
  resolution: "Update VPC DHCP options to include AD DNS server IPs. Verify DNS port 53 is open."

- symptoms: "Access denied joining domain"
  diagnosis: "The AD service account lacks permission to create computer objects in the specified OU."
  resolution: "Grant the service account CreateComputer and ResetPassword permissions on the target OU."

- symptoms: "File system created but authentication fails"
  diagnosis: "AD trust relationship issue or Kerberos port (88) blocked."
  resolution: "Verify security group allows TCP/UDP 88. Check AD trust if using a resource forest."

## Output Format

```yaml
root_cause: "ad_integration — <specific_cause>"
evidence:
  - type: file_system
    content: "<describe-file-systems with AD configuration and FailureDetails>"
  - type: directory
    content: "<directory service configuration>"
  - type: network
    content: "<security group rules and DNS configuration>"
severity: HIGH
mitigation:
  immediate: "Fix AD connectivity, DNS, or permissions and recreate file system"
  long_term: "Standardize AD integration with validated network and permission templates"
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
