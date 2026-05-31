---
title: "G2 — ElastiCache Restore Issues"
description: "Diagnose issues restoring Redis clusters from snapshots"
status: active
severity: HIGH
triggers:
  - "restore failed"
  - "restore snapshot"
  - "create from backup"
  - "snapshot restore"
  - "seed cluster"
owner: devops-agent
objective: "Successfully restore a Redis cluster from a snapshot"
context: "Redis clusters can be restored from snapshots to create new clusters or seed new replication groups. The target node type must have sufficient memory for the snapshot data. Restoring creates a new cluster — it does not overwrite an existing one. Snapshots can be restored to different node types, engine versions (upgrade only), and configurations. Memcached does not support restore (no snapshots)."
---

## Phase 1 — Triage

MUST:
- Check snapshot details: `aws elasticache describe-snapshots --snapshot-name <snapshot-name>`
- Verify snapshot status is 'available': `aws elasticache describe-snapshots --snapshot-name <snapshot-name> --query 'Snapshots[*].SnapshotStatus'`
- Check snapshot size and source configuration: `aws elasticache describe-snapshots --snapshot-name <snapshot-name> --query 'Snapshots[*].{NodeType:CacheNodeType,Engine:Engine,EngineVersion:EngineVersion,NumNodeGroups:NumNodeGroups}'`
- Check recent events: `aws elasticache describe-events --source-type cache-cluster --duration 1440`
- Verify target node type has sufficient memory for the snapshot data

SHOULD:
- Check if the target engine version is compatible (can upgrade, not downgrade)
- Verify subnet group and security group exist for the target cluster
- Check if the snapshot is from a cluster mode enabled or disabled cluster
- Verify KMS key access if the snapshot is encrypted

MAY:
- Check if the snapshot was exported to S3: `aws elasticache describe-snapshots --snapshot-name <snapshot-name> --query 'Snapshots[*].{Bucket:TopicArn}'`
- Verify service quotas for the target cluster configuration
- Check if parameter group is compatible with the target engine version

## Phase 2 — Remediate

MUST:
- Restore to a node type with sufficient memory: `aws elasticache create-replication-group --replication-group-id <new-id> --replication-group-description "<desc>" --snapshot-name <snapshot-name> --cache-node-type <node-type> --cache-subnet-group-name <subnet-group> --security-group-ids <sg-id>`
- Ensure the target engine version is equal to or higher than the snapshot version
- Configure security groups and subnet groups for the new cluster

SHOULD:
- Test the restore in a non-production environment first
- Update application connection strings to point to the new cluster
- Verify data integrity after restore

MAY:
- Restore with different cluster configuration (e.g., add replicas, change node type)
- Import an RDB file from S3 for migration: `aws elasticache create-replication-group --snapshot-arns <s3-arn>`
- Set up monitoring on the restored cluster

## Common Issues

- symptoms: "Restore fails with InsufficientCacheClusterCapacity"
  diagnosis: "Target node type is not available in the requested AZ."
  resolution: "Try a different node type or AZ. Check AWS capacity in the region."

- symptoms: "Restore fails — incompatible engine version"
  diagnosis: "Target engine version is lower than the snapshot's engine version."
  resolution: "Restore to the same or higher engine version. Downgrade is not supported."

- symptoms: "Restored cluster has no data"
  diagnosis: "Snapshot was from an empty cluster or the wrong snapshot was used."
  resolution: "Verify the snapshot name and check its size. Use describe-snapshots to confirm data."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Restore to new cluster from snapshot | GREEN | Creates new cluster; no impact on existing infrastructure |
| Configure security groups on restored cluster | GREEN | Network configuration for new cluster |
| Update application connection strings | YELLOW | Application change; must coordinate with all consumers |
| Test restore in non-production | GREEN | Verification step; no production impact |

## Escalation Conditions

- Restore failure on a critical snapshot needed for disaster recovery
- InsufficientCacheClusterCapacity preventing restore in target AZ
- Incompatible engine version between snapshot and target cluster
- Encrypted snapshot with inaccessible KMS key
- Restore needed as part of active disaster recovery procedure

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-snapshots` | LOW | Snapshot metadata only |
| `describe-events` | LOW | Operational events only |
| Restored cluster data | HIGH | Contains actual cached data from the snapshot |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) on the restored cluster
- NEVER suggest disabling AUTH on the restored cluster if the source had AUTH enabled
- NEVER suggest disabling encryption in transit on the restored cluster
- NEVER suggest reducing node count during peak traffic
- NEVER restore a snapshot to a node type with insufficient memory for the data

## Phase 3 — Rollback

If restore causes issues:
1. If the restored cluster has issues, delete it and retry with different parameters: `aws elasticache delete-replication-group --replication-group-id <restored-id>`
2. If application was pointed to the restored cluster and it has issues, revert connection strings to the original cluster
3. If the wrong snapshot was restored, delete the cluster and restore from the correct snapshot
4. If engine version is incompatible, restore to a cluster with the correct engine version
5. Verify data integrity on the restored cluster before routing production traffic

## Output Format

```yaml
root_cause: "restore_issue — <specific_cause>"
evidence:
  - type: snapshot_details
    content: "<snapshot configuration and status>"
  - type: target_config
    content: "<target cluster configuration>"
  - type: events
    content: "<restore-related events>"
severity: HIGH
mitigation:
  immediate: "Fix restore parameters and retry"
  long_term: "Document restore procedures and test regularly"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
