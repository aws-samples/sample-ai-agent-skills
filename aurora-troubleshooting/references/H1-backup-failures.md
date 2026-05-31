---
title: "H1 — Backup Failures"
description: "Diagnose Aurora cluster backup and snapshot failures"
status: active
severity: HIGH
triggers:
  - "backup failed"
  - "snapshot failed"
  - "automated backup"
  - "manual snapshot"
  - "backup retention"
  - "backup window"
owner: devops-agent
objective: "Identify and resolve Aurora backup and snapshot failures"
context: "Aurora automated backups are continuous and incremental, stored in S3 (managed by AWS). Manual snapshots are user-initiated. Backup retention is 1-35 days. Aurora backups are cluster-level (not instance-level). Backup failures can stem from KMS key issues, service limits, or cluster state."
---

## Phase 1 — Triage

MUST:
- Check cluster backup configuration:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].{BackupRetention:BackupRetentionPeriod,BackupWindow:PreferredBackupWindow,LatestRestore:LatestRestorableTime}'
  ```
- Check cluster snapshots:
  ```
  aws rds describe-db-cluster-snapshots --db-cluster-identifier <cluster-id> \
    --query 'DBClusterSnapshots[].{Id:DBClusterSnapshotIdentifier,Status:Status,Created:SnapshotCreateTime,Type:SnapshotType}'
  ```
- Check cluster events for backup-related issues:
  ```
  aws rds describe-events --source-identifier <cluster-id> --source-type db-cluster --duration 1440 \
    --event-categories backup
  ```
- Check cluster status: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Status'`

SHOULD:
- Check KMS key status if cluster is encrypted:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].KmsKeyId'
  aws kms describe-key --key-id <kms-key-id>
  ```
- Check snapshot quotas: `aws service-quotas get-service-quota --service-code rds --quota-code L-272F1212`
- Verify backup retention period is set: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].BackupRetentionPeriod'`

MAY:
- Check CloudTrail for snapshot API calls
- Review cross-region snapshot copy configuration if applicable

## Phase 2 — Remediate

MUST:
- For KMS key issues: ensure the KMS key is enabled and the RDS service has access
- For snapshot quota exceeded: delete old manual snapshots or request quota increase
- For cluster in error state: resolve the underlying cluster issue first

SHOULD:
- Set backup retention to at least 7 days for production clusters
- Configure backup window during low-traffic periods
- Enable cross-region snapshot copy for disaster recovery

MAY:
- Automate snapshot management with AWS Backup
- Implement snapshot lifecycle policies

## Common Issues

- symptoms: "Manual snapshot creation failed"
  diagnosis: "Cluster in modifying state, KMS key disabled, or snapshot quota exceeded."
  resolution: "Wait for cluster to be available. Check KMS key. Delete old snapshots."

- symptoms: "LatestRestorableTime is stale"
  diagnosis: "Automated backup may be delayed or cluster has issues."
  resolution: "Check cluster events. Verify cluster is in available state."

- symptoms: "Cross-region snapshot copy failed"
  diagnosis: "KMS key not available in target region or insufficient permissions."
  resolution: "Create a KMS key in the target region. Grant cross-region permissions."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-cluster-snapshots, describe-events, CloudWatch backup metrics, describe-key — read-only inspection
- YELLOW: create-db-cluster-snapshot, modify-db-cluster (backup retention/window) — recoverable changes
- RED: delete-db-cluster-snapshot, delete-db-cluster — destructive operations, potential data loss

## Escalation Conditions
- "Database serves production traffic"
- "Backup failures affecting recovery point objectives (RPO)"
- "KMS key issues preventing backup or restore operations"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, KMS key IDs and policies, snapshot ARNs (expose encryption configuration)
- HIGH: CloudTrail events for snapshot operations (contain API caller identity)
- MEDIUM: backup configuration, retention periods, snapshot status

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix backup issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest deleting manual snapshots without confirming they are no longer needed"
- "NEVER suggest reducing backup retention below compliance requirements"

## Phase 3 — Rollback
- "Restore from snapshot if backup configuration change causes issues"
- "If KMS key was disabled, re-enable immediately to restore backup functionality"
- "Revert backup retention and window changes if they impact production performance"

## Output Format

```yaml
root_cause: "backup_failure — <specific_cause>"
evidence:
  - type: cluster_config
    content: "<backup configuration>"
  - type: snapshots
    content: "<snapshot status>"
  - type: events
    content: "<backup-related events>"
severity: HIGH
mitigation:
  immediate: "Resolve backup failure cause"
  long_term: "Implement backup monitoring and AWS Backup integration"
```

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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
