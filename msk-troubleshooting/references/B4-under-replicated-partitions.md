---
title: "B4 — MSK Under-Replicated Partitions"
description: "Diagnose and resolve under-replicated partitions in MSK"
status: active
severity: CRITICAL
triggers:
  - "under-replicated"
  - "UnderReplicatedPartitions"
  - "ISR shrink"
  - "replica lag"
  - "out of sync"
  - "UnderMinIsrPartitionCount"
owner: devops-agent
objective: "Identify why partitions are under-replicated and restore full replication"
context: "Under-replicated partitions mean follower replicas are not in sync with the leader. This is a critical health indicator — if the leader fails while replicas are behind, data loss can occur. Causes include broker overload (CPU, disk, network), broker failure, uneven partition distribution, large messages, or insufficient replication throughput. The UnderReplicatedPartitions metric should always be 0 in a healthy cluster."
---

## Phase 1 — Triage

MUST:
- Check UnderReplicatedPartitions per broker: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name UnderReplicatedPartitions --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check UnderMinIsrPartitionCount: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name UnderMinIsrPartitionCount --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check per-broker CPU: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name CpuUser --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check per-broker disk usage: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name KafkaDataLogsDiskUsed --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- List broker nodes: `aws kafka list-nodes --cluster-arn <cluster-arn>`

SHOULD:
- Check topic details: `kafka-topics.sh --bootstrap-server <broker>:9092 --describe --under-replicated-partitions`
- Check ReplicaFetcherManager metrics
- Check network throughput per broker
- Check for ongoing cluster operations: `aws kafka list-cluster-operations --cluster-arn <cluster-arn>`

MAY:
- Check broker logs for replication errors
- Check disk I/O latency metrics
- Review num.replica.fetchers configuration

## Phase 2 — Remediate

MUST:
- If broker CPU > 80%: scale instance type or redistribute partitions
- If broker disk full: increase storage or reduce retention (see A3)
- If broker failed: wait for automatic replacement, then rebalance
- If network saturated: scale instance type for more network bandwidth

SHOULD:
- Increase num.replica.fetchers if replication throughput is the bottleneck
- Rebalance partitions to distribute load evenly across brokers
- Ensure all brokers have similar resource utilization

MAY:
- Increase replica.fetch.max.bytes for large messages
- Consider reducing replication factor for non-critical topics
- Review message size — very large messages slow replication

## Common Issues

- symptoms: "UnderReplicatedPartitions on all brokers during rolling restart"
  diagnosis: "Expected during configuration updates or Kafka version upgrades."
  resolution: "Monitor until rolling restart completes. Partitions should return to fully replicated."

- symptoms: "UnderReplicatedPartitions on a single broker consistently"
  diagnosis: "That broker is overloaded (CPU, disk, or network) and cannot keep replicas in sync."
  resolution: "Scale the broker instance type or move partitions to other brokers."

- symptoms: "UnderMinIsrPartitionCount > 0"
  diagnosis: "Partitions have fewer in-sync replicas than min.insync.replicas. Writes with acks=all will fail."
  resolution: "Critical — restore broker health immediately. This blocks producers using acks=all."

## Output Format

```yaml
root_cause: "under_replicated_partitions — <specific_cause>"
evidence:
  - type: replication_metrics
    content: "<UnderReplicatedPartitions, UnderMinIsrPartitionCount per broker>"
  - type: broker_metrics
    content: "<CPU, disk, network per broker>"
  - type: topic_details
    content: "<under-replicated partition list>"
severity: CRITICAL
mitigation:
  immediate: "Resolve broker bottleneck to restore replication"
  long_term: "Balance partitions, right-size brokers, monitor replication metrics"
```


## Safety Ratings
```
safety_ratings:
  - "Check UnderReplicatedPartitions metrics: GREEN — read-only monitoring query"
  - "List broker nodes: GREEN — read-only API call"
  - "Describe under-replicated partitions: GREEN — read-only Kafka CLI command"
  - "Scale broker instance type: RED — triggers rolling restart, temporary partition unavailability"
  - "Redistribute partitions: YELLOW — moves data between brokers, impacts throughput during transfer"
  - "Increase num.replica.fetchers: YELLOW — broker configuration change triggers rolling restart"
```

## Escalation Conditions
- Cluster serves production event streaming
- UnderMinIsrPartitionCount > 0 (producers with acks=all will fail)
- Under-replication persists beyond a rolling restart window
- Multiple brokers showing under-replicated partitions simultaneously
- Data loss risk if leader broker fails while replicas are behind

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Partition assignment details: topic and broker mapping"
    - "Replication metrics: cluster health indicators"
    - "Broker resource utilization: capacity data"
  handling: "Do not expose partition assignment or replication topology externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER reduce replication factor to resolve under-replication (masks the problem)
- NEVER force-reassign partitions during active broker instability
- NEVER ignore UnderMinIsrPartitionCount > 0 — this is a data loss risk

## Phase 3 — Rollback
- If partition reassignment was executed: re-run reassignment with original assignment plan to restore previous distribution
- If broker instance type was scaled: scale back during maintenance window if original type was sufficient
- If num.replica.fetchers was increased: revert broker configuration and apply (triggers rolling restart)
- If retention was reduced to free disk space: restore original retention.ms after storage is expanded
- If replica.fetch.max.bytes was changed: revert to default value if not needed

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"
