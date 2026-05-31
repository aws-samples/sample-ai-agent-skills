---
title: "B1 — MSK Producer Throughput Issues"
description: "Diagnose and resolve low producer throughput or producer failures"
status: active
severity: HIGH
triggers:
  - "producer slow"
  - "low throughput"
  - "producer timeout"
  - "BytesInPerSec"
  - "ProduceRequestsPerSec"
  - "batch size"
  - "producer performance"
owner: devops-agent
objective: "Identify bottlenecks in producer throughput and optimize for target performance"
context: "Producer throughput depends on batch size, linger.ms, compression, acks setting, partition count, broker capacity, and network. Common bottlenecks include acks=all with high min.insync.replicas, insufficient partitions, small batch sizes, no compression, broker CPU/disk saturation, or network bandwidth limits. MSK broker throughput is bounded by instance type."
---

## Phase 1 — Triage

MUST:
- Check BytesInPerSec per broker: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name BytesInPerSec --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check broker CPU: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name CpuUser --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check broker disk usage: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name KafkaDataLogsDiskUsed --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check ProduceMessageConversionsPerSec (message format conversion overhead): `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name ProduceMessageConversionsPerSec --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check UnderReplicatedPartitions: indicates broker overload
- Check topic partition count: `kafka-topics.sh --bootstrap-server <broker>:9092 --describe --topic <topic>`
- Review producer configuration (batch.size, linger.ms, compression.type, acks, buffer.memory)
- Check network throughput per broker

MAY:
- Check RequestHandlerAvgIdlePercent (broker thread saturation)
- Check NetworkProcessorAvgIdlePercent
- Review producer-side metrics (record-send-rate, record-error-rate, request-latency-avg)

## Phase 2 — Remediate

MUST:
- If broker CPU > 80%: scale to larger instance type or add brokers
- If insufficient partitions: increase partition count for the topic (cannot decrease)
- If acks=all with min.insync.replicas = replication factor: reduce min.insync.replicas by 1

SHOULD:
- Optimize producer batch.size (default 16384, try 65536-131072 for throughput)
- Set linger.ms > 0 (e.g., 5-20ms) to allow batching
- Enable compression (lz4 or snappy for throughput, zstd for compression ratio)
- Increase buffer.memory if producers are blocking on full buffers

MAY:
- Use multiple producer instances for parallel throughput
- Consider partitioning strategy to distribute load evenly
- Enable idempotent producer (enable.idempotence=true) for exactly-once without throughput penalty

## Common Issues

- symptoms: "Producer throughput much lower than expected"
  diagnosis: "Small batch.size and linger.ms=0 causing one-message-per-request."
  resolution: "Increase batch.size to 65536+ and set linger.ms to 5-20ms."

- symptoms: "Producer timeouts with acks=all"
  diagnosis: "min.insync.replicas equals replication factor; one slow replica blocks all writes."
  resolution: "Set min.insync.replicas = replication_factor - 1 (e.g., 2 for RF=3)."

- symptoms: "Uneven throughput across brokers"
  diagnosis: "Partition leader imbalance — most partitions led by a few brokers."
  resolution: "Run preferred leader election or rebalance partitions across brokers."

## Output Format

```yaml
root_cause: "producer_throughput — <specific_cause>"
evidence:
  - type: cloudwatch_metrics
    content: "<BytesInPerSec, CpuUser, disk usage per broker>"
  - type: topic_config
    content: "<partition count, replication factor>"
  - type: producer_config
    content: "<batch.size, linger.ms, acks, compression>"
severity: HIGH
mitigation:
  immediate: "Optimize producer configuration or scale brokers"
  long_term: "Right-size partitions, enable compression, monitor throughput metrics"
```


## Safety Ratings
```
safety_ratings:
  - "Check CloudWatch metrics: GREEN — read-only monitoring query"
  - "Check topic partition count: GREEN — read-only Kafka CLI command"
  - "Optimize producer batch.size and linger.ms: GREEN — client-side configuration change"
  - "Increase partition count: YELLOW — irreversible, cannot decrease partitions"
  - "Scale broker instance type: RED — triggers rolling restart, temporary partition unavailability"
  - "Add brokers: YELLOW — adds capacity, requires partition rebalancing for benefit"
```

## Escalation Conditions
- Cluster serves production event streaming
- Producer throughput degradation impacting downstream consumers
- Broker CPU consistently above 80% across multiple brokers
- Fix requires instance type change (rolling restart) during business hours
- Throughput issues causing message backlog in upstream systems

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Producer configuration: batch.size, linger.ms, acks settings"
    - "Topic partition counts: capacity planning data"
    - "Throughput metrics: business activity volume indicators"
  handling: "Do not expose producer configuration details or throughput patterns externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER set acks=0 on production topics handling critical data
- NEVER reduce min.insync.replicas below 2 for production topics with RF=3
- NEVER increase partition count without understanding consumer group impact

## Phase 3 — Rollback
- If producer batch.size or linger.ms was changed: revert client configuration to previous values and redeploy producers
- If partition count was increased: CANNOT be rolled back — partitions cannot be decreased
- If broker instance type was scaled: scale back to previous instance type during maintenance window (triggers another rolling restart)
- If min.insync.replicas was changed: restore original value via kafka-configs.sh --alter
- If compression was enabled: disable compression in producer config and redeploy (may increase network usage)

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
