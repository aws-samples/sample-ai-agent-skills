---
title: "H2 — ElastiCache Resharding Failures"
description: "Diagnose failed or stuck resharding operations in Redis cluster mode enabled"
status: active
severity: HIGH
triggers:
  - "resharding failed"
  - "resharding stuck"
  - "shard configuration"
  - "modify-replication-group-shard-configuration"
  - "resharding error"
owner: devops-agent
objective: "Resolve resharding failures and successfully complete shard configuration changes"
context: "Resharding modifies the number of shards in a cluster mode enabled replication group. It can add shards (scale out) or remove shards (scale in). The operation moves hash slots between shards. Failures can occur due to insufficient capacity, memory constraints, ongoing operations, or configuration limits. The replication group must be in 'available' status to start resharding."
---

## Phase 1 — Triage

MUST:
- Check replication group status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{Status:Status,ClusterEnabled:ClusterEnabled}'`
- Check recent events for errors: `aws elasticache describe-events --source-type replication-group --duration 1440`
- Check current shard count and configuration: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups'`
- Check pending modifications: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].PendingModifiedValues'`
- Verify cluster mode is enabled (resharding requires CME)

SHOULD:
- Check if another modification is in progress
- Verify the target shard count is within limits (max 500 nodes total across all shards and replicas)
- Check memory usage per shard: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check CloudTrail for the resharding API call and any errors

MAY:
- Check service quotas for ElastiCache nodes
- Verify subnet group has IPs available for new nodes
- Check if the engine version supports online resharding

## Phase 2 — Remediate

MUST:
- Ensure replication group is in 'available' status before retrying
- Wait for any in-progress modifications to complete
- Verify cluster mode is enabled (CMD cannot be resharded)
- Retry resharding: `aws elasticache modify-replication-group-shard-configuration --replication-group-id <repl-group-id> --node-group-count <target> --apply-immediately`

SHOULD:
- Plan resharding during low-traffic periods
- Ensure sufficient capacity in target AZs for new nodes
- Monitor the operation via describe-events and describe-replication-groups

MAY:
- Request service quota increase if node limits are reached
- Consider a phased approach (add one shard at a time) for large changes
- Test resharding in non-production first

## Common Issues

- symptoms: "Resharding fails with InvalidReplicationGroupState"
  diagnosis: "Replication group is not in 'available' status. Another operation is in progress."
  resolution: "Wait for the current operation to complete. Check status with describe-replication-groups."

- symptoms: "InsufficientCacheClusterCapacity during resharding"
  diagnosis: "Not enough capacity for new nodes in the target AZs."
  resolution: "Try different AZs or node types. Check AWS capacity in the region."

- symptoms: "Cannot reduce shard count below data requirements"
  diagnosis: "Remaining shards cannot hold all the data from removed shards."
  resolution: "Ensure remaining shards have sufficient memory. Reduce data size before scaling in."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Wait for resharding to complete | GREEN | No action; monitoring only |
| Retry resharding | YELLOW | Restarts slot migration; MOVED/ASK redirections during operation |
| Request service quota increase | GREEN | Administrative action; no operational impact |
| Plan phased approach (one shard at a time) | GREEN | Reduces risk per operation |

## Escalation Conditions

- Resharding failed mid-operation leaving cluster in inconsistent state
- InvalidReplicationGroupState preventing any modifications
- InsufficientCacheClusterCapacity in target AZs
- Resharding required during active production incident
- Cannot reduce shard count because remaining shards lack memory

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-replication-groups` (status, shards) | MEDIUM | Exposes cluster architecture |
| `describe-events` | LOW | Operational events only |
| `get-metric-statistics` (memory per shard) | LOW | Operational metrics only |
| `lookup-events` (CloudTrail) | MEDIUM | Exposes API calls and IAM principals |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) before resharding
- NEVER suggest disabling AUTH during resharding
- NEVER suggest disabling encryption in transit during resharding
- NEVER suggest reducing node count during peak traffic
- NEVER force-cancel a resharding operation (may leave slots in inconsistent state)

## Phase 3 — Rollback

If resharding failure causes issues:
1. If resharding failed, wait for the cluster to return to 'available' status before retrying
2. If shards were added and cause issues, remove them with another resharding operation
3. If shards were removed and data doesn't fit, add shards back immediately
4. If the cluster is stuck in a non-available state, contact AWS Support
5. Monitor cluster status, slot distribution, and ReplicationLag after rollback

## Output Format

```yaml
root_cause: "resharding_failure — <specific_cause>"
evidence:
  - type: replication_group
    content: "<status and shard configuration>"
  - type: events
    content: "<resharding error events>"
  - type: capacity
    content: "<node count and limits>"
severity: HIGH
mitigation:
  immediate: "Resolve the blocking condition and retry resharding"
  long_term: "Plan resharding with capacity checks and during low-traffic windows"
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
