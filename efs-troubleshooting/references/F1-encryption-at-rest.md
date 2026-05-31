---
title: "F1 â€” EFS Encryption at Rest Issues"
description: "Diagnose encryption at rest configuration and KMS key issues"
status: active
severity: MEDIUM
triggers:
  - "encryption at rest"
  - "KMS key EFS"
  - "encrypted file system"
  - "unencrypted EFS"
owner: devops-agent
objective: "Identify encryption at rest issues and guide migration if needed"
context: "EFS encryption at rest uses KMS keys. It must be enabled at creation time and cannot be changed. The KMS key must be accessible to the EFS service. Using a customer-managed key allows cross-account access control."
---

## Phase 1 â€” Triage

MUST:
- Check encryption status: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].Encrypted'`
- Check KMS key if encrypted: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].KmsKeyId'`
- Verify KMS key is enabled: `aws kms describe-key --key-id <key-id>`

SHOULD:
- Check KMS key policy allows EFS service
- Verify KMS key is not scheduled for deletion

MAY:
- Check if the KMS key has key rotation enabled
- Verify cross-account KMS key access if applicable

## Phase 2 â€” Remediate

MUST:
- If file system needs encryption: create a new encrypted file system and migrate data
- If KMS key is disabled: re-enable it
- If KMS key is deleted: restore from pending deletion if within grace period

SHOULD:
- Use AWS DataSync for data migration between file systems
- Enable KMS key rotation for security

MAY:
- Switch from AWS-managed key to customer-managed key (requires new file system)

## Common Issues

- symptoms: "Need to encrypt an existing unencrypted file system"
  diagnosis: "Encryption at rest cannot be added after creation."
  resolution: "Create a new encrypted file system and migrate data using DataSync or rsync."

- symptoms: "EFS operations fail with KMS error"
  diagnosis: "KMS key is disabled, deleted, or the key policy doesn't allow EFS."
  resolution: "Re-enable the KMS key. Ensure the key policy grants EFS service access."

- symptoms: "Cross-account access fails on encrypted file system"
  diagnosis: "The KMS key policy doesn't allow the other account."
  resolution: "Update KMS key policy to allow the cross-account principal."

## Output Format

```yaml
root_cause: "encryption_at_rest_issue â€” <specific_cause>"
evidence:
  - type: encryption_status
    content: "<file system encryption configuration>"
  - type: kms_key_status
    content: "<KMS key state>"
severity: MEDIUM
mitigation:
  immediate: "Fix KMS key or plan migration to encrypted file system"
  long_term: "Ensure all new file systems are created with encryption enabled"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — EFS provides shared storage for multiple compute instances |
| Data Risk | CRITICAL — File system contains application data and shared state |
| Recovery Complexity | HIGH — Data loss from file system deletion is unrecoverable without backups |
| Blast Radius | HIGH — Mount or performance issues affect all connected instances |

## Escalation Conditions

- File system unavailable or mount failures affecting multiple production instances
- Throughput degradation causing application timeouts across the fleet
- Burst credits depleted with no provisioned throughput configured
- Security group changes blocking NFS access for running workloads
- Replication lag exceeding RPO thresholds for disaster recovery
- IAM authorization failures preventing application access to file system

## Data Sensitivity

- File system contents may include application data, configuration files, and shared secrets
- Access point configurations reveal application directory structure and POSIX permissions
- Mount target IPs and DNS names are internal infrastructure identifiers
- Encryption keys (KMS) protect data at rest — key access must be tightly controlled
- EFS access logs may contain file paths revealing business logic or data organization

## PROHIBITED Actions

- NEVER suggest deleting a file system with active mount targets
- NEVER remove mount targets from availability zones with running instances
- NEVER change security group rules without verifying all connected instance groups
- NEVER disable encryption in transit on file systems handling sensitive data
- NEVER modify access point POSIX permissions without understanding application requirements
- NEVER switch throughput modes during peak usage without capacity planning

## Phase 3 — Rollback

- If mount target was removed: recreate in the same subnet with the same security groups
- If security group rules were changed: restore previous inbound TCP 2049 rules from backup
- If throughput mode was changed: revert to previous mode and monitor burst credit balance
- If access point permissions were modified: restore original POSIX UID/GID and directory path
- If lifecycle policy was changed: revert transition rules and verify IA storage class files
- Document all rollback actions with file system ID, mount target IDs, and timestamps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing IAM authorization"
  - "NEVER suggest overly broad file system policies"
