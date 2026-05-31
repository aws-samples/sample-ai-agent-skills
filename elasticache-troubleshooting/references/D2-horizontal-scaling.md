---
title: "D2 — ElastiCache Horizontal Scaling (Resharding)"
description: "Diagnose issues with adding or removing shards in Redis cluster mode enabled"
status: active
severity: HIGH
triggers:
  - "resharding"
  - "add shards"
  - "remove shards"
  - "horizontal scaling"
  - "scale out"
  - "scale in"
owner: devops-agent
objective: "Successfully complete horizontal scaling operations and resolve resharding failures"
context: "Horizontal scaling (resharding) adds or removes shards in a cluster mode enabled Redis replication group. Online resharding moves hash slots between shards without downtime. During resharding, multi-key operations on migrating slots may receive MOVED/ASK errors. Resharding large shards can take hours. Cluster mode disabled does NOT support resharding — it has only one shard."
---

## Phase 1 — Triage

MUST:
- Check cluster mode status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{ClusterEnabled:ClusterEnabled,Status:Status}'`
- Check current shard configuration: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups'`
- Check for pending modifications: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].PendingModifiedValues'`
- Check recent events: `aws elasticache describe-events --source-type replication-group --duration 1440`
- Check ReplicationLag during resharding: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

SHOULD:
- Verify the cluster is in 'available' status before resharding
- Check memory usage per shard to plan slot distribution
- Verify the target shard count is within limits (max 500 nodes total)
- Check if any previous resharding operation failed

MAY:
- Review slot distribution: `redis-cli -h <config-endpoint> -p 6379 CLUSTER SLOTS`
- Check for unbalanced slot distribution: `redis-cli -h <config-endpoint> -p 6379 CLUSTER INFO`
- Monitor network throughput during resharding

## Phase 2 — Remediate

MUST:
- Ensure cluster mode is enabled (CMD cannot be resharded)
- Use modify-replication-group-shard-configuration for resharding: `aws elasticache modify-replication-group-shard-configuration --replication-group-id <repl-group-id> --node-group-count <target-count> --apply-immediately`
- Ensure clients handle MOVED and ASK redirections during resharding
- Monitor the operation via describe-events

SHOULD:
- Perform resharding during low-traffic periods to minimize impact
- Balance slot distribution evenly across shards
- Monitor ReplicationLag and EngineCPU during resharding
- Test resharding in non-production first

MAY:
- Use custom slot distribution for specific data locality requirements
- Consider ElastiCache Serverless for automatic scaling without manual resharding
- Plan capacity for the additional nodes before resharding

## Common Issues

- symptoms: "Resharding stuck at a percentage for hours"
  diagnosis: "Large shard with many keys — slot migration is slow."
  resolution: "Wait for completion. Large shards with millions of keys can take hours. Monitor progress via events."

- symptoms: "CROSSSLOT error during resharding"
  diagnosis: "Multi-key command operating on keys in different slots, one of which is migrating."
  resolution: "Ensure multi-key operations use hash tags {tag} to keep keys in the same slot."

- symptoms: "Cannot reshard — cluster mode is disabled"
  diagnosis: "Cluster mode disabled (CMD) has a single shard and cannot be resharded."
  resolution: "Create a new cluster mode enabled cluster and migrate data. CMD cannot be converted to CME in-place."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Ensure cluster mode enabled | GREEN | Verification step; no change |
| Add shards (scale out) | YELLOW | Online resharding; MOVED/ASK redirections during migration |
| Remove shards (scale in) | RED | Data redistribution; risk of data loss if remaining shards lack memory |
| Ensure clients handle MOVED/ASK | GREEN | Application-level verification |
| Balance slot distribution | YELLOW | Slot migration; transient errors during redistribution |

## Escalation Conditions

- Resharding operation on a production cluster with >100GB data
- Resharding stuck or failed mid-operation
- MOVED/ASK errors causing application failures during resharding
- Scale-in operation when remaining shards may not have sufficient memory
- Resharding required during active production incident

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-replication-groups` (shard config) | MEDIUM | Exposes cluster architecture |
| `describe-events` | LOW | Operational events only |
| `get-metric-statistics` (ReplicationLag) | LOW | Operational metrics only |
| `CLUSTER SLOTS` | LOW | Slot distribution only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) before resharding
- NEVER suggest disabling AUTH during resharding
- NEVER suggest disabling encryption in transit during resharding
- NEVER suggest reducing node count during peak traffic
- NEVER start resharding while another modification is in progress

## Phase 3 — Rollback

If horizontal scaling causes issues:
1. If shards were added and cause issues, remove them: `aws elasticache modify-replication-group-shard-configuration --replication-group-id <repl-group-id> --node-group-count <previous-count> --apply-immediately` (note: lengthy operation)
2. If shards were removed and data doesn't fit, add shards back immediately
3. If resharding is stuck, contact AWS Support — do not attempt to force cancel
4. If MOVED/ASK errors persist after resharding completes, verify client library handles redirections
5. Monitor slot distribution and ReplicationLag after rollback

## Output Format

```yaml
root_cause: "horizontal_scaling — <specific_cause>"
evidence:
  - type: cluster_config
    content: "<shard count and cluster mode status>"
  - type: events
    content: "<resharding events>"
  - type: slot_distribution
    content: "<current slot allocation>"
severity: HIGH
mitigation:
  immediate: "Resolve resharding failure or wait for completion"
  long_term: "Plan resharding during low-traffic windows, ensure client handles redirections"
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
