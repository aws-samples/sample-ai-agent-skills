---
title: "I2 — Restore Issues"
description: "Diagnose and resolve Redshift snapshot restore failures"
status: active
severity: HIGH
triggers:
  - "restore failed"
  - "restore from snapshot"
  - "restore cluster"
  - "snapshot restore"
owner: devops-agent
objective: "Successfully restore a Redshift cluster from snapshot"
context: "Restoring from a snapshot creates a new cluster with the data from the snapshot point-in-time. The new cluster can have different configuration (node type, count, VPC). Restore time depends on data volume and node type. RA3 restores are faster due to managed storage."
---

## Phase 1 — Triage

MUST:
- Check snapshot availability: `aws redshift describe-cluster-snapshots --snapshot-identifier <snapshot-id> --query 'Snapshots[0].{status:Status,size:TotalBackupSizeInMegaBytes,encrypted:Encrypted}'`
- Check restore status: `aws redshift describe-clusters --cluster-identifier <new-cluster-id> --query 'Clusters[0].{status:ClusterStatus,restoreStatus:RestoreStatus}'`
- Verify target configuration is valid (node type, count, VPC, subnet group)
- Check CloudTrail for RestoreFromClusterSnapshot errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RestoreFromClusterSnapshot --max-results 5`

SHOULD:
- Verify KMS key is available if snapshot is encrypted
- Check target VPC/subnet group exists and has available IPs
- Verify the new cluster identifier doesn't already exist

MAY:
- Check if restoring cross-region snapshot (requires KMS key in target region)
- Verify IAM permissions for restore operation
- Check service quotas for cluster limits

## Phase 2 — Remediate

MUST:
- If snapshot not available: check if it was deleted or expired
- If KMS key error: ensure KMS key is enabled and accessible in target region
- If VPC/subnet error: create or fix subnet group configuration
- If cluster ID conflict: use a different cluster identifier

SHOULD:
- Restore to same node type for fastest restore time
- Configure security groups and parameter groups after restore
- Update application connection strings to new cluster endpoint

MAY:
- Use table restore for partial data recovery
- Restore to Serverless namespace from provisioned snapshot
- Implement automated restore testing

## Common Issues

- symptoms: "Restore fails with 'KMS key not found'"
  diagnosis: "Encrypted snapshot requires KMS key in the restore region."
  resolution: "Copy KMS key to target region or use cross-region snapshot copy with destination KMS key."

- symptoms: "Restore takes hours for large cluster"
  diagnosis: "DC2 restore requires full data copy. RA3 restore is faster (data in managed storage)."
  resolution: "Expected for DC2. Consider RA3 for faster restore times."

## Output Format

```yaml
root_cause: "Restore issue — <specific_cause>"
evidence:
  - type: snapshot_status
    content: "<snapshot and restore status>"
severity: HIGH
mitigation:
  immediate: "Fix KMS, VPC, or configuration issue and retry restore"
  long_term: "Use RA3 for faster restores, test restore procedures regularly"
```


## Safety Ratings
```
safety_ratings:
  - "Check snapshot availability: GREEN — read-only API call"
  - "Check restore status: GREEN — read-only API call"
  - "Check CloudTrail for restore errors: GREEN — read-only audit log query"
  - "Restore from snapshot: RED — creates new cluster, requires connection string updates and validation"
  - "Table restore: YELLOW — restores specific tables, may overwrite existing data"
```

## Escalation Conditions
- Cluster serves production analytics
- Fix requires resize or restore
- Restore needed for disaster recovery
- KMS key not available in target region
- Restore taking longer than expected for DC2 nodes
- Application connection strings need updating after restore

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Snapshot data: full cluster data including all tables"
    - "KMS key configuration: encryption details for restore"
    - "New cluster endpoint: connection details"
    - "User credentials: database access information preserved in snapshot"
  handling: "Restored clusters contain all original data. Apply same access controls as the original cluster."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER restore to a cluster identifier that already exists
- NEVER delete the source snapshot before confirming the restore completed successfully
- NEVER skip security group and parameter group configuration after restore

## Phase 3 — Rollback
- If restore created a cluster with wrong configuration: delete the restored cluster and re-restore with correct settings
- If table restore overwrote data: restore the table again from a different snapshot point
- If restored cluster has wrong security groups: update security groups to match production requirements
- If application connection strings were updated: revert to original cluster endpoint if restore is abandoned

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
