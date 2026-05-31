---
title: "J1 — Global Datastore Cross-Region Replication Lag"
description: "Diagnose high replication lag in ElastiCache Global Datastore for Redis"
status: active
severity: HIGH
triggers:
  - "Global Datastore"
  - "cross-region replication"
  - "global replication lag"
  - "secondary region lag"
  - "global datastore lag"
owner: devops-agent
objective: "Identify and reduce cross-region replication lag in Global Datastore"
context: "ElastiCache Global Datastore enables cross-region replication for Redis with cluster mode enabled. Typical replication lag is under 1 second. The primary region handles all writes; secondary regions are read-only. High lag can be caused by heavy write volume, network latency between regions, large key operations, or secondary region capacity issues. Requires Redis 5.0.6+ and cluster mode enabled."
---

## Phase 1 — Triage

MUST:
- Check Global Datastore status: `aws elasticache describe-global-replication-groups --global-replication-group-id <global-id>`
- Check cross-region replication lag: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<secondary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum,Average --region <secondary-region>`
- Check primary region write volume: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name SetTypeCmds --dimensions Name=CacheClusterId,Value=<primary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum --region <primary-region>`
- Check secondary region EngineCPU: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<secondary-cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum --region <secondary-region>`
- Check recent events in both regions: `aws elasticache describe-events --source-type replication-group --duration 1440 --region <region>`

SHOULD:
- Check network throughput between regions
- Verify secondary region cluster has the same node type as primary
- Check if resharding is in progress on either region
- Check memory usage on secondary region nodes

MAY:
- Check AWS Health Dashboard for cross-region connectivity issues
- Compare shard count between primary and secondary
- Review if large batch writes are causing replication bursts

## Phase 2 — Remediate

MUST:
- Ensure secondary region has the same node type and shard count as primary
- Reduce write volume on primary if lag is consistently high
- Scale up secondary region nodes if EngineCPU is saturated

SHOULD:
- Distribute writes across shards evenly (avoid hot shards)
- Monitor GlobalDatastoreReplicationLag with CloudWatch alarms
- Avoid large batch writes that create replication bursts

MAY:
- Consider reducing the number of secondary regions if lag is unacceptable
- Optimize write patterns to reduce replication stream size
- Evaluate if the application can tolerate eventual consistency for cross-region reads

## Common Issues

- symptoms: "Cross-region replication lag exceeding 5 seconds"
  diagnosis: "Heavy write volume on primary exceeds secondary's ability to apply changes."
  resolution: "Scale up secondary nodes. Reduce write volume. Distribute writes across shards."

- symptoms: "Replication lag spikes during peak hours"
  diagnosis: "Write volume spikes cause temporary replication backlog."
  resolution: "Scale secondary to handle peak write volume. Implement write throttling if possible."

- symptoms: "Secondary region shows stale data"
  diagnosis: "Cross-region replication lag means secondary reads return older data."
  resolution: "This is expected with async replication. For strong consistency, read from primary region."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Ensure secondary has same node type as primary | YELLOW | May require scaling secondary; brief impact |
| Distribute writes across shards | GREEN | Application-level optimization |
| Monitor with CloudWatch alarms | GREEN | Monitoring only; no operational impact |
| Scale up secondary region nodes | YELLOW | Node type change; causes failover in secondary |
| Reduce number of secondary regions | RED | Removes cross-region replication; data loss in removed region |

## Escalation Conditions

- Cross-region replication lag exceeding 5 seconds in production
- Secondary region unable to keep up with primary write volume
- Replication lag causing stale reads impacting customer experience
- AWS service health issue affecting cross-region connectivity
- Resharding in progress on either region affecting replication

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-global-replication-groups` | MEDIUM | Exposes multi-region architecture |
| `get-metric-statistics` (ReplicationLag) | LOW | Operational metrics only |
| `get-metric-statistics` (SetTypeCmds) | LOW | Operational metrics only |
| `describe-events` (both regions) | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to reduce replication lag
- NEVER suggest disabling AUTH on any region
- NEVER suggest disabling encryption in transit on any region
- NEVER suggest reducing node count during peak traffic
- NEVER remove a secondary region without confirming no applications depend on it for reads

## Phase 3 — Rollback

If replication lag remediation causes issues:
1. If secondary node type was changed, revert to previous type
2. If write patterns were modified, revert application code
3. If a secondary region was removed, re-add it (note: full data sync required)
4. If CloudWatch alarms are generating noise, adjust thresholds
5. Monitor ReplicationLag in all secondary regions after rollback

## Output Format

```yaml
root_cause: "global_datastore_lag — <specific_cause>"
evidence:
  - type: replication_lag
    content: "<cross-region ReplicationLag metrics>"
  - type: write_volume
    content: "<primary region write metrics>"
  - type: secondary_cpu
    content: "<secondary region EngineCPU>"
severity: HIGH
mitigation:
  immediate: "Scale secondary region or reduce primary write volume"
  long_term: "Right-size both regions, implement lag monitoring and alerting"
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
