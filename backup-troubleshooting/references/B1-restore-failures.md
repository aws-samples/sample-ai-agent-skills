---
title: "B1 — Restore Failures"
description: "Diagnose why AWS Backup restore jobs are failing"
status: active
severity: HIGH
triggers:
  - "restore failed"
  - "restore error"
  - "cannot restore"
  - "restore job failed"
  - "recovery point restore"
owner: devops-agent
objective: "Identify the root cause of restore job failures and successfully restore from backup"
context: "Restore operations create new resources from recovery points. Failures can occur due to IAM permissions, KMS key issues, resource conflicts (e.g., duplicate names), subnet/VPC configuration, or service-specific constraints. The restore job StatusMessage provides the most diagnostic value."
---

## Phase 1 — Triage

MUST:
- List failed restore jobs: `aws backup list-restore-jobs --by-status FAILED --max-results 20`
- Describe the failed restore job: `aws backup describe-restore-job --restore-job-id <job-id>`
- Check the StatusMessage for the specific error
- Verify the recovery point exists and is in COMPLETED status: `aws backup describe-recovery-point --backup-vault-name <vault> --recovery-point-arn <rp-arn>`
- Check the IAM role used for the restore

SHOULD:
- Verify the restore metadata is correct (subnet, security group, instance type, etc.)
- Check KMS key permissions for the recovery point's encryption key
- Verify the target VPC/subnet exists and has available IP addresses
- Check for resource name conflicts (e.g., RDS instance identifier already exists)

MAY:
- Check CloudTrail for the StartRestoreJob event details
- Verify service quotas are not exceeded (e.g., max RDS instances, max EBS volumes)
- Check if the recovery point is in cold storage (restore from cold takes longer)

## Phase 2 — Remediate

MUST:
- Fix the identified issue (permissions, metadata, resource conflicts)
- Ensure the IAM role has restore permissions for the specific service
- Retry the restore with correct parameters

SHOULD:
- Use unique resource names/identifiers to avoid conflicts
- Verify VPC and subnet configuration before restoring network-attached resources
- Ensure KMS key is available and the restore role has decrypt permissions

MAY:
- Use restore testing to validate restorability before critical restore needs
- Document restore procedures and required metadata for each resource type

## Common Issues

- symptoms: "Access Denied during restore"
  diagnosis: "The restore IAM role lacks service-specific permissions (e.g., ec2:RunInstances, rds:CreateDBInstance)."
  resolution: "Attach AWSBackupServiceRolePolicyForRestores or add specific service permissions."

- symptoms: "DBInstanceAlreadyExists error for RDS restore"
  diagnosis: "An RDS instance with the same identifier already exists."
  resolution: "Specify a unique DBInstanceIdentifier in the restore metadata."

- symptoms: "Restore from cold storage fails or takes very long"
  diagnosis: "Recovery points in cold storage require retrieval before restore (up to several hours)."
  resolution: "Allow sufficient time for cold storage retrieval. Plan ahead for cold storage restores."

- symptoms: "Subnet or security group not found during restore"
  diagnosis: "The restore metadata references a subnet or security group that no longer exists."
  resolution: "Update restore metadata with valid subnet and security group IDs."

## Output Format

```yaml
root_cause: "restore_failure — <specific_cause>"
evidence:
  - type: restore_job
    content: "<describe-restore-job output with StatusMessage>"
  - type: recovery_point
    content: "<recovery point status and details>"
  - type: restore_metadata
    content: "<metadata configuration>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry the restore"
  long_term: "Implement restore testing and document restore procedures"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix permissions, metadata, resource conflicts | YELLOW | Configuration changes — review each |
| Ensure IAM role has restore permissions | YELLOW | IAM policy change |
| Retry restore with correct parameters | YELLOW | Creates new resources — verify parameters |
| Use unique resource names | GREEN | Naming guidance — non-destructive |
| Verify VPC/subnet configuration | GREEN | Verification only |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- Restore operations create new resources that may conflict with existing ones
- Cross-account restore requires coordination between accounts

## Data Sensitivity

- Recovery points contain full resource data — restore creates copies of potentially sensitive data
- Restore metadata reveals infrastructure configuration
- VPC/subnet configuration reveals network architecture

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER restore to production without verifying restore parameters
- NEVER restore sensitive data to less-secure environments

## Phase 3 — Rollback

- If restore created incorrect resource: delete the restored resource (new resource, not original)
- If IAM permissions were changed: revert to previous policy
- If restore metadata was incorrect: retry with corrected parameters
- Document all restore operations for compliance audit trail

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
