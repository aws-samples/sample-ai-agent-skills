---
title: "A1 — Backup Job Failures"
description: "Diagnose why AWS Backup jobs are failing"
status: active
severity: HIGH
triggers:
  - "backup job failed"
  - "backup failure"
  - "backup error"
  - "job status FAILED"
  - "backup not completing"
owner: devops-agent
objective: "Identify the root cause of backup job failures and restore successful backup operations"
context: "AWS Backup jobs can fail for many reasons including IAM permission issues, KMS key problems, resource state conflicts, service limits, or network connectivity. The backup job status message and CloudTrail events provide the most diagnostic value."
---

## Phase 1 — Triage

MUST:
- List recent failed backup jobs: `aws backup list-backup-jobs --by-state FAILED --max-results 20`
- Describe the specific failed job: `aws backup describe-backup-job --backup-job-id <job-id>`
- Check the StatusMessage field for the specific error reason
- Check the ResourceType and ResourceArn to identify what was being backed up
- Verify the IAM role used: `aws iam get-role --role-name <role-from-job>`

SHOULD:
- Check CloudTrail for the failure event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=backup.amazonaws.com --max-results 20`
- Verify the resource exists and is in a valid state
- Check if the backup vault exists and is accessible: `aws backup describe-backup-vault --backup-vault-name <vault-name>`
- Check KMS key status: `aws kms describe-key --key-id <key-id>`

MAY:
- Check CloudWatch metrics for backup failure trends: `aws cloudwatch get-metric-statistics --namespace AWS/Backup --metric-name NumberOfBackupJobsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check for service limit issues: `aws backup describe-region-settings`
- Review backup vault notifications for alert configuration

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (IAM permissions, KMS key policy, resource state)
- Retry the backup job after fixing the issue
- Verify the IAM role has the required service-specific permissions (e.g., ec2:CreateSnapshot for EBS)

SHOULD:
- Set up SNS notifications on the backup vault for future failure alerts
- Review and update the backup plan if the failure is configuration-related
- Ensure the KMS key policy grants the backup role encrypt/decrypt permissions

MAY:
- Enable AWS Backup Audit Manager to monitor backup compliance
- Create CloudWatch alarms for backup job failures

## Common Issues

- symptoms: "Access Denied error in backup job StatusMessage"
  diagnosis: "The IAM role lacks required permissions for the target service."
  resolution: "Attach the AWS managed policy AWSBackupServiceRolePolicyForBackup or add specific service permissions."

- symptoms: "KMS key error — unable to encrypt/decrypt"
  diagnosis: "The backup role does not have permissions on the KMS key, or the key is disabled/pending deletion."
  resolution: "Update KMS key policy to grant the backup role kms:GenerateDataKey, kms:Decrypt, kms:CreateGrant."

- symptoms: "Resource not found or invalid state"
  diagnosis: "The resource was deleted, is in a transitional state, or the ARN is incorrect."
  resolution: "Verify the resource exists and is in an available/running state before backup."

- symptoms: "Backup job fails for RDS instance in modifying state"
  diagnosis: "RDS instance is being modified (scaling, patching) and cannot be snapshotted."
  resolution: "Wait for the RDS instance to return to 'available' state before retrying."

## Output Format

```yaml
root_cause: "backup_job_failure — <specific_cause>"
evidence:
  - type: backup_job
    content: "<describe-backup-job output with StatusMessage>"
  - type: iam_role
    content: "<role permissions analysis>"
  - type: kms_key
    content: "<key status and policy>"
severity: HIGH
mitigation:
  immediate: "Fix the identified permission/configuration issue and retry"
  long_term: "Set up monitoring and alerts for backup job failures"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix IAM permissions, KMS key policy, resource state | YELLOW | Permission/configuration changes — review scope |
| Retry the backup job | GREEN | Re-executes backup — non-destructive |
| Verify IAM role has service-specific permissions | GREEN | Verification only |
| Set up SNS notifications on vault | GREEN | Monitoring — non-destructive |
| Review and update backup plan | YELLOW | Changes backup behavior |
| Enable Backup Audit Manager | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- KMS key policy changes affect encryption for all recovery points in the vault
- IAM role changes affect backup operations across multiple resource types

## Data Sensitivity

- Backup job details reveal which resources are protected and their backup status
- KMS key policies control encryption of backup data at rest
- IAM role configurations reveal backup access patterns
- CloudTrail events may contain sensitive resource identifiers

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible) — this is impossible after cooling-off period
- NEVER suggest reducing retention below compliance requirements
- NEVER delete recovery points without understanding retention obligations
- NEVER modify KMS keys used by backup vaults without understanding encryption dependencies

## Phase 3 — Rollback

- If IAM permissions were changed: revert IAM policy to previous version
- If KMS key policy was modified: restore original key policy
- If backup plan was updated: restore previous plan version using `aws backup update-backup-plan`
- If SNS notifications were configured incorrectly: update or remove notification configuration
- Document all changes for compliance audit trail

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
