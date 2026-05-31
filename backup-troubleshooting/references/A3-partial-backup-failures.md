---
title: "A3 — Partial Backup Failures"
description: "Diagnose why some resources in a backup plan succeed while others fail"
status: active
severity: MEDIUM
triggers:
  - "partial backup"
  - "some backups failed"
  - "not all resources backed up"
  - "missing backup"
  - "incomplete backup"
owner: devops-agent
objective: "Identify why specific resources within a backup plan fail while others succeed"
context: "When a backup plan targets multiple resources (via tags or ARNs), individual resource backups are independent jobs. Some may succeed while others fail due to resource-specific issues like permissions, state, or service limitations."
---

## Phase 1 — Triage

MUST:
- List all backup jobs for the plan's time window: `aws backup list-backup-jobs --by-backup-plan-id <plan-id> --by-created-after <start> --by-created-before <end>`
- Identify which jobs failed vs succeeded by comparing ResourceArn values
- Describe each failed job: `aws backup describe-backup-job --backup-job-id <job-id>`
- Check the backup selection: `aws backup get-backup-selection --backup-plan-id <plan-id> --selection-id <selection-id>`

SHOULD:
- Compare IAM permissions for the failed resource types vs successful ones
- Check if failed resources have different KMS encryption keys
- Verify failed resources are in a valid state (available, running)
- Check if failed resources are in a different region or account than expected

MAY:
- Check if the resource type is supported by AWS Backup
- Verify tag-based selection is matching the expected resources: `aws resourcegroupstaggingapi get-resources --tag-filters Key=<key>,Values=<value>`
- Review CloudTrail for resource-specific errors

## Phase 2 — Remediate

MUST:
- Fix resource-specific issues (permissions, state, encryption)
- Ensure the IAM role has permissions for ALL resource types in the selection
- Retry failed backups after fixing issues

SHOULD:
- Create separate backup selections for resources with different permission requirements
- Ensure all KMS keys grant the backup role appropriate permissions
- Add SNS notifications to catch partial failures early

MAY:
- Use AWS Backup Audit Manager to monitor backup coverage
- Create CloudWatch alarms for per-resource-type backup failures

## Common Issues

- symptoms: "EC2 backups succeed but EFS backups fail"
  diagnosis: "The backup role has EC2 permissions but lacks EFS permissions."
  resolution: "Add AWSBackupServiceRolePolicyForBackup managed policy or specific EFS permissions."

- symptoms: "Resources with customer-managed KMS keys fail, default-encrypted succeed"
  diagnosis: "The backup role lacks permissions on the customer-managed KMS key."
  resolution: "Update the KMS key policy to grant the backup role kms:CreateGrant and kms:Decrypt."

- symptoms: "Tag-based selection misses some resources"
  diagnosis: "Resources have slightly different tag values (case-sensitive) or tags were added after selection."
  resolution: "Verify tag values match exactly. Tag-based selection is case-sensitive for StringEquals."

## Output Format

```yaml
root_cause: "partial_backup_failure — <specific_cause>"
evidence:
  - type: failed_jobs
    content: "<list of failed jobs with StatusMessage>"
  - type: successful_jobs
    content: "<list of successful jobs for comparison>"
  - type: selection
    content: "<backup selection configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix resource-specific issues and retry failed backups"
  long_term: "Ensure consistent permissions and monitoring across all resource types"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix resource-specific issues (permissions, state, encryption) | YELLOW | Per-resource changes — review each |
| Ensure IAM role has permissions for ALL resource types | YELLOW | IAM policy change — review scope |
| Retry failed backups | GREEN | Re-executes backup — non-destructive |
| Create separate backup selections | YELLOW | Changes backup plan structure |
| Add SNS notifications | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Fix requires modifying vault lock (IRREVERSIBLE in compliance mode)
- Fix requires cross-account operations affecting multiple AWS accounts
- IAM role changes affect backup operations across multiple resource types
- KMS key policy changes affect encryption for multiple resources

## Data Sensitivity

- Partial failure analysis reveals which resources are and aren't protected
- Tag-based selection reveals resource classification and organization
- KMS key configurations reveal encryption strategy per resource type

## Prohibited Actions

- NEVER suggest deleting vault lock in compliance mode (irreversible)
- NEVER suggest reducing retention below compliance requirements
- NEVER ignore partial failures — unprotected resources represent compliance gaps
- NEVER modify backup selections without understanding resource coverage impact

## Phase 3 — Rollback

- If IAM permissions were changed: revert to previous IAM policy
- If backup selections were modified: restore original selection configuration
- If KMS key policies were changed: restore original key policies
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
