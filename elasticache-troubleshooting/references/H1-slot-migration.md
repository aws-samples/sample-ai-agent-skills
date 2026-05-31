---
title: "H1 — ElastiCache Cluster Mode Slot Migration Issues"
description: "Diagnose hash slot migration problems in Redis cluster mode enabled"
status: active
severity: HIGH
triggers:
  - "slot migration"
  - "MOVED"
  - "ASK"
  - "hash slot"
  - "migrating slot"
  - "importing slot"
owner: devops-agent
objective: "Resolve slot migration issues and ensure data is correctly distributed across shards"
context: "Redis cluster mode enabled uses 16,384 hash slots distributed across shards. Slot migration moves slots between shards during resharding. During migration, clients may receive MOVED (permanent redirect) or ASK (temporary redirect) responses. Clients must handle these redirections. Slot migration is an online operation but can cause transient errors for multi-key operations on keys in migrating slots."
---

## Phase 1 — Triage

MUST:
- Check cluster slot distribution: `redis-cli -h <config-endpoint> -p 6379 CLUSTER SLOTS`
- Check cluster info: `redis-cli -h <config-endpoint> -p 6379 CLUSTER INFO`
- Check for migrating/importing slots: `redis-cli -h <node-endpoint> -p 6379 CLUSTER NODES | grep -E 'migrating|importing'`
- Check replication group status: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{Status:Status,NodeGroups:NodeGroups}'`
- Check recent events: `aws elasticache describe-events --source-type replication-group --duration 1440`

SHOULD:
- Check if resharding is in progress: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].Status'`
- Monitor ReplicationLag during migration: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check for unbalanced slot distribution across shards
- Verify client handles MOVED/ASK redirections

MAY:
- Check individual node cluster state: `redis-cli -h <node-endpoint> -p 6379 CLUSTER INFO | grep cluster_state`
- Monitor network throughput during migration
- Check for stuck migrations (slots in migrating state for extended periods)

## Phase 2 — Remediate

MUST:
- Ensure Redis Cluster-aware client library is used (handles MOVED/ASK automatically)
- Wait for in-progress resharding to complete before starting new operations
- If migration is stuck, check events and consider canceling/retrying

SHOULD:
- Use hash tags {tag} for related keys that need multi-key operations
- Monitor slot migration progress via describe-events
- Plan resharding during low-traffic periods

MAY:
- Manually rebalance slots if distribution is uneven
- Check if specific slots have disproportionately many keys
- Consider using CLUSTER KEYSLOT to verify key-to-slot mapping

## Common Issues

- symptoms: "MOVED errors in application logs"
  diagnosis: "Client is sending commands to the wrong node for the key's hash slot."
  resolution: "Use a Redis Cluster-aware client that handles MOVED redirections automatically."

- symptoms: "Slots stuck in migrating state"
  diagnosis: "Resharding operation failed or is extremely slow due to large key count."
  resolution: "Check events for errors. If stuck, contact AWS Support. Large migrations can take hours."

- symptoms: "Uneven slot distribution after resharding"
  diagnosis: "Custom slot distribution was specified or resharding did not complete fully."
  resolution: "Use modify-replication-group-shard-configuration to rebalance slots evenly."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Use Redis Cluster-aware client | GREEN | Application-level improvement; handles redirections |
| Wait for resharding to complete | GREEN | No action; monitoring only |
| Use hash tags for related keys | GREEN | Application-level key naming convention |
| Monitor slot migration progress | GREEN | Monitoring only; no operational impact |

## Escalation Conditions

- Slots stuck in migrating state for more than 2 hours
- MOVED/ASK errors causing application failures in production
- Uneven slot distribution after resharding completion
- Slot migration causing excessive replication lag
- Application not handling MOVED/ASK redirections

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `CLUSTER SLOTS` | LOW | Slot distribution only |
| `CLUSTER INFO` | LOW | Cluster state only |
| `CLUSTER NODES` | MEDIUM | Exposes node IDs, endpoints, and slot assignments |
| `describe-replication-groups` | MEDIUM | Exposes cluster architecture |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix slot migration issues
- NEVER suggest disabling AUTH during slot migration
- NEVER suggest disabling encryption in transit during slot migration
- NEVER suggest reducing node count during peak traffic
- NEVER manually manipulate slot assignments using CLUSTER commands on managed ElastiCache

## Phase 3 — Rollback

If slot migration causes issues:
1. If resharding is in progress and causing problems, wait for completion — interrupting may leave slots in inconsistent state
2. If slot distribution is uneven after resharding, rebalance: `aws elasticache modify-replication-group-shard-configuration --replication-group-id <repl-group-id> --node-group-count <count> --apply-immediately`
3. If application cannot handle MOVED/ASK, switch to a Redis Cluster-aware client library
4. If stuck migrations are detected, contact AWS Support
5. Monitor CLUSTER INFO for cluster_state:ok after rollback

## Output Format

```yaml
root_cause: "slot_migration — <specific_cause>"
evidence:
  - type: cluster_slots
    content: "<slot distribution>"
  - type: cluster_nodes
    content: "<migrating/importing slots>"
  - type: events
    content: "<resharding events>"
severity: HIGH
mitigation:
  immediate: "Wait for migration to complete or fix client redirection handling"
  long_term: "Use hash tags for related keys, plan resharding during low-traffic"
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
