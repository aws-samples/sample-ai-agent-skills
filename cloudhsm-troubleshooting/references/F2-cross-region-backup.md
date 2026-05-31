---
title: "F2 — Cross-Region Backup"
description: "Diagnose and resolve CloudHSM cross-region backup and restore issues"
status: active
severity: HIGH
triggers:
  - "cross-region backup"
  - "backup copy failed"
  - "cluster restore"
  - "backup retention"
  - "disaster recovery"
owner: devops-agent
objective: "Resolve cross-region backup failures and ensure disaster recovery readiness for CloudHSM"
context: "CloudHSM automatically creates daily backups of the cluster. Backups can be copied to other regions for disaster recovery. Cross-region backup copy requires proper IAM permissions and the destination region must support CloudHSM. Backups contain all keys, users, and configuration. Restoring from backup creates a new cluster with the same cryptographic material."
---

## Phase 1 — Triage

MUST:
- List recent backups: `aws cloudhsmv2 describe-backups --filters clusterIds=<cluster-id> --query 'Backups[*].{ID:BackupId,State:BackupState,CreateTime:CreateTimestamp}'`
- Check if cross-region copy was attempted: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CopyBackupToRegion --max-results 10`
- Verify IAM permissions for cross-region copy: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names cloudhsm:CopyBackupToRegion`
- Check backup state in the destination region: `aws cloudhsmv2 describe-backups --region <dest-region> --query 'Backups[*].{ID:BackupId,State:BackupState,SourceCluster:SourceCluster}'`

SHOULD:
- Verify the destination region supports CloudHSM
- Check if there are any service quota limits on backups in the destination region
- Verify the backup retention policy: `aws cloudhsmv2 describe-backups --filters clusterIds=<cluster-id> --query 'Backups[*].{ID:BackupId,DeleteTimestamp:DeleteTimestamp}'`

MAY:
- Review the backup schedule and retention settings
- Check if automated cross-region copy is configured (via Lambda or Step Functions)
- Verify the backup encryption key configuration

## Phase 2 — Remediate

MUST:
- If cross-region copy fails with permission error: add `cloudhsm:CopyBackupToRegion` to the IAM policy
- If backup is in DELETED state: use a different backup or create a new one manually
- If destination region has no backups: initiate a copy: `aws cloudhsmv2 copy-backup-to-region --destination-region <region> --backup-id <backup-id>`
- If restore fails: verify the backup is in READY state and the VPC/subnet configuration is valid in the target region

SHOULD:
- Set up automated cross-region backup copy using CloudWatch Events and Lambda
- Verify the restored cluster has all expected keys and users: `listUsers` and `findKey`
- Test disaster recovery procedures periodically

MAY:
- Implement backup lifecycle policies to manage retention and costs
- Create runbooks for full DR failover procedures
- Set up monitoring for backup age and cross-region copy status

## Common Issues

- symptoms: "CopyBackupToRegion returns AccessDeniedException"
  diagnosis: "IAM policy missing cloudhsm:CopyBackupToRegion permission."
  resolution: "Add the permission to the IAM policy. Also ensure the policy allows the destination region."

- symptoms: "Backup shows CREATE_IN_PROGRESS for extended time"
  diagnosis: "Large cluster with many keys — backup creation takes longer."
  resolution: "Wait for completion. Backups typically complete within minutes."

- symptoms: "Restored cluster missing keys that existed in the source"
  diagnosis: "Backup was taken before the keys were created."
  resolution: "Use a more recent backup. Verify backup timestamp against key creation time."

- symptoms: "Cannot restore backup — cluster creation from backup fails"
  diagnosis: "VPC/subnet configuration in the target region is invalid or missing."
  resolution: "Create the required VPC, subnets, and security groups in the target region before restoring."

## Output Format

```yaml
root_cause: "Cross-region backup issue — <specific_cause>"
evidence:
  - type: backup_status
    content: "<describe-backups output>"
  - type: copy_events
    content: "<CloudTrail events for backup operations>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or configuration and retry backup copy"
  long_term: "Automate cross-region backup with monitoring and periodic DR testing"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/list): GREEN — read-only"
  - "Phase 2 backup copy: YELLOW — creates resources in another region"
  - "Phase 2 cluster restore from backup: YELLOW — creates new billable cluster"

## Escalation Conditions

escalation_conditions:
  - "No recent backups exist for a production cluster"
  - "Cross-region copy fails repeatedly with internal errors"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Disaster recovery is needed and backups are unavailable"
  - "Backup integrity is in question"

## Data Sensitivity

data_sensitivity:
  - command: "describe-backups"
    sensitivity: HIGH
    contains: "Backup IDs, cluster IDs, and timestamps"
  - command: "copy-backup-to-region"
    sensitivity: HIGH
    contains: "Cross-region transfer of encrypted key material"
  - command: "cloudtrail lookup-events"
    sensitivity: HIGH
    contains: "API call details and principal ARNs"

## Prohibited Actions

prohibited_actions:
  - "NEVER delete backups without verifying newer backups exist"
  - "NEVER restore a backup to a region without proper network and security configuration"
  - "NEVER share backup IDs or cluster details with unauthorized parties"
  - "NEVER disable automatic backups on production clusters"
  - "NEVER skip verification of restored cluster contents"

## Phase 3 — Rollback

- If a cross-region copy created an unwanted backup, delete it in the destination region
- If a cluster was restored from backup incorrectly, delete the restored cluster
- If IAM policy changes were too broad, scope them down after the copy succeeds
- Document the backup ID and region used for any restore operation
- Verify the source cluster is unaffected by any backup or restore operations
