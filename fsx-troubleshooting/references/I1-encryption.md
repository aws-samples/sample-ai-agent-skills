---
title: "I1 — Encryption Issues"
description: "Diagnose encryption configuration and KMS key problems on FSx"
status: active
severity: HIGH
triggers:
  - "encryption error"
  - "KMS key"
  - "encryption at rest"
  - "key access denied"
  - "decrypt error"
owner: devops-agent
objective: "Identify and resolve encryption-related issues on FSx file systems"
context: "All FSx file systems are encrypted at rest using KMS keys specified at creation. The key cannot be changed after creation. Issues typically involve KMS key access, key deletion/disabling, cross-account key access, or key policy misconfiguration."
---

## Phase 1 — Triage

MUST:
- Describe the file system for KMS key: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check KMS key status: `aws kms describe-key --key-id <key-id>`
- Verify KMS key policy grants FSx access: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Check if the key is enabled and not pending deletion

SHOULD:
- Verify KMS key grants: `aws kms list-grants --key-id <key-id>`
- Check if the FSx service-linked role has KMS permissions
- Review CloudTrail for KMS access denied events
- Verify key is in the same region as the file system

MAY:
- Check for key rotation status: `aws kms get-key-rotation-status --key-id <key-id>`
- Verify cross-account KMS access if applicable
- Review KMS key usage metrics

## Phase 2 — Remediate

MUST:
- Re-enable the KMS key if it was disabled
- Cancel key deletion if the key is pending deletion
- Update KMS key policy to grant FSx service principal access

SHOULD:
- Use customer-managed KMS keys for production file systems
- Enable automatic key rotation for customer-managed keys
- Document KMS key dependencies for FSx file systems

MAY:
- Set up CloudWatch alarms for KMS key state changes
- Implement key policy templates for FSx deployments

## Common Issues

- symptoms: "File system enters FAILED state after KMS key disabled"
  diagnosis: "The KMS key used for encryption was disabled or scheduled for deletion."
  resolution: "Re-enable the KMS key or cancel deletion. The file system should recover automatically."

- symptoms: "Cannot create file system with customer-managed KMS key"
  diagnosis: "KMS key policy does not grant fsx.amazonaws.com the required permissions."
  resolution: "Add kms:CreateGrant, kms:Decrypt, kms:DescribeKey, kms:GenerateDataKey* to the key policy for fsx.amazonaws.com."

- symptoms: "Want to change the encryption key on existing file system"
  diagnosis: "The encryption key is immutable after file system creation."
  resolution: "Create a new file system with the desired KMS key and migrate data from the old file system."

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: kms_key
    content: "<key status, policy, and grants>"
  - type: file_system
    content: "<file system encryption configuration>"
  - type: cloudtrail
    content: "<KMS access denied events>"
severity: HIGH
mitigation:
  immediate: "Fix KMS key status or policy"
  long_term: "Implement KMS key management best practices and monitoring"
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
