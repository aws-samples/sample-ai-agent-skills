---
name: msk-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon MSK (Managed
  Streaming for Apache Kafka) and MSK Serverless problems by analyzing
  cluster configurations, broker health, producer/consumer performance,
  connectivity, authentication, topic management, MSK Connect, security,
  monitoring, and Serverless capacity issues.
  Activate when: cluster creation failures, broker failures, storage full,
  producer throughput degradation, consumer lag, high latency, under-
  replicated partitions, client connection failures, authentication errors,
  VPC/security group issues, topic creation errors, partition rebalancing,
  retention issues, MSK Connect connector failures or scaling, IAM auth
  problems, SASL/SCRAM issues, TLS/mTLS errors, missing CloudWatch metrics,
  Prometheus open monitoring, MSK Serverless cluster issues, Serverless
  capacity/throughput limits, or the user says something is wrong with
  Kafka/MSK without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MSK (kafka), CloudWatch, CloudTrail,
  EC2 (for VPC/security groups), IAM, Secrets Manager (for SASL/SCRAM),
  and optionally KMS permissions. Apache Kafka CLI tools recommended for
  direct cluster diagnostics.
---

# MSK Diagnostics

## When to use

Any MSK investigation where the console alone is insufficient — cluster health debugging, broker failure analysis, producer/consumer performance tuning, connectivity troubleshooting, authentication resolution, topic management, MSK Connect issues, security configuration, monitoring setup, or Serverless capacity problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws kafka list-clusters-v2
aws kafka describe-cluster-v2 --cluster-arn <cluster-arn>
aws kafka list-nodes --cluster-arn <cluster-arn>
aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name ActiveControllerCount --dimensions Name=Cluster\ Name,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Minimum
aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name OfflinePartitionsCount --dimensions Name=Cluster\ Name,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum
aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name KafkaDataLogsDiskUsed --dimensions Name=Cluster\ Name,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Cluster deep dive

```
aws kafka describe-cluster-operation --cluster-operation-arn <operation-arn>
aws kafka list-cluster-operations --cluster-arn <cluster-arn>
aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>
aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name CpuUser --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum
aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name UnderReplicatedPartitions --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 3 — Detailed investigation

```
aws kafka list-configurations
aws kafka describe-configuration --arn <config-arn>
aws kafka describe-configuration-revision --arn <config-arn> --revision <revision>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kafka.amazonaws.com --max-results 20
aws kafka list-scram-secrets --cluster-arn <cluster-arn>
```

Read `references/msk-guardrails.md` before concluding on any MSK issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-clusters-v2` | List all MSK clusters (provisioned and Serverless) |
| `describe-cluster-v2` | Full cluster configuration and status |
| `list-nodes` | Broker node details (instance type, storage, endpoints) |
| `get-bootstrap-brokers` | Bootstrap broker connection strings for all auth methods |
| `list-cluster-operations` | Recent cluster operations (updates, scaling) |
| `describe-cluster-operation` | Details of a specific cluster operation |
| `describe-configuration` | Kafka broker configuration properties |
| `list-scram-secrets` | SASL/SCRAM secrets associated with the cluster |
| `list-client-vpc-connections` | VPC connections for multi-VPC connectivity |
| `CloudWatch AWS/Kafka` | Broker, topic, and consumer group metrics |
| `kafka-topics.sh` | Topic management (list, describe, create, alter) |
| `kafka-consumer-groups.sh` | Consumer group lag and offset management |

## Gotchas: Amazon MSK

- Broker storage is NOT auto-scaling by default (must enable): MSK provisioned clusters do not automatically scale broker storage. You must explicitly enable auto-scaling via the console, CLI, or API. Without it, brokers will run out of disk and become unhealthy. Storage auto-scaling has a cooldown period and scales in 10 GiB increments. Once storage is increased, it cannot be decreased.
- Replication factor vs min.insync.replicas: The replication factor (set per topic) determines how many copies of each partition exist. `min.insync.replicas` (set in broker or topic config) determines how many replicas must acknowledge a write for it to succeed when `acks=all`. Setting `min.insync.replicas` equal to the replication factor means a single broker failure blocks all writes to that topic.
- Consumer group lag indicates processing delays: Consumer lag (the difference between the latest offset and the consumer's committed offset) is the primary indicator of consumer health. High lag means consumers cannot keep up with producers. Check `MaxOffsetLag` and `SumOffsetLag` CloudWatch metrics per consumer group. Lag can be caused by slow processing, insufficient consumers, or rebalancing.
- Partition count is immutable (can only increase, never decrease): Once a topic is created with N partitions, you can only increase the partition count, never decrease it. Increasing partitions changes key-based message ordering guarantees. Plan partition count carefully based on expected throughput. A common guideline is max(T/P, T/C) where T=throughput, P=producer throughput per partition, C=consumer throughput per partition.
- MSK Connect vs self-managed connectors: MSK Connect is a managed Kafka Connect service. It uses custom plugins (uploaded to S3) and manages worker infrastructure. Self-managed connectors run on your own EC2/ECS. MSK Connect has different scaling behavior (autoscaled or provisioned capacity) and different logging/monitoring. Not all community connectors are compatible without modification.
- IAM vs SASL/SCRAM vs mTLS authentication: MSK supports three auth methods. IAM auth uses AWS SigV4 and requires IAM policies. SASL/SCRAM uses username/password stored in Secrets Manager. mTLS uses client certificates from ACM Private CA. Each method uses different ports (9098 for IAM, 9096 for SASL/SCRAM, 9094 for TLS). Mixing auth methods on the same cluster is supported but each client must use the correct port and protocol.
- MSK Serverless has different limits than provisioned: MSK Serverless automatically scales but has limits: max 5 consumer groups per partition, max 120 MB/s write and 360 MB/s read per cluster, max 200 partitions per topic, max 50 topics. Serverless uses IAM auth only. No broker configuration customization. Different pricing model (pay per data in/out and partition-hours).
- Topic-level configs override broker-level defaults: Topic-level configurations (retention.ms, max.message.bytes, cleanup.policy, etc.) override the corresponding broker-level defaults. When troubleshooting retention or message size issues, always check both topic-level and broker-level configs. Use `kafka-configs.sh --describe --topic <topic>` to see topic-level overrides.
- Under-replicated partitions indicate broker issues: The `UnderReplicatedPartitions` metric is a key health indicator. Non-zero values mean some partition replicas are not in sync with the leader. Causes include broker overload, network issues, disk I/O bottlenecks, or broker failures. Sustained under-replication risks data loss if the leader fails.
- Storage scaling is one-way (cannot shrink broker volumes): Once you increase broker storage volume size, you cannot decrease it. Storage auto-scaling only scales up. Plan initial storage carefully. Use topic retention policies and log compaction to manage data growth. The minimum storage per broker is 1 GiB and maximum is 16384 GiB (16 TiB).

## Anti-hallucination rules

1. Always cite specific cluster configurations, broker node status, CloudWatch metrics, or Kafka CLI output as evidence.
2. Never claim MSK broker storage auto-scales by default. Auto-scaling must be explicitly enabled.
3. Never suggest decreasing partition count or broker storage volume size. Both are one-way operations.
4. Never confuse IAM auth port (9098), SASL/SCRAM port (9096), and TLS port (9094). Each auth method has a specific port.
5. MSK Serverless uses IAM auth only and has different limits — never apply provisioned cluster concepts (broker count, instance types, custom configs) to Serverless.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 28 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A3 | Creation failures, broker failures, storage full |
| B — Performance | B1-B4 | Producer throughput, consumer lag, high latency, under-replicated partitions |
| C — Connectivity | C1-C3 | Client connection failures, authentication issues, VPC/security group |
| D — Topics | D1-D3 | Topic creation errors, partition rebalancing, retention issues |
| E — MSK Connect | E1-E2 | Connector failures, connector scaling |
| F — Security | F1-F3 | IAM auth, SASL/SCRAM, TLS/mTLS |
| G — Monitoring | G1-G2 | Missing metrics, open monitoring (Prometheus) |
| H — Serverless | H1-H2 | Serverless cluster issues, capacity/throughput |
| Z — Catch-All | Z1 | General troubleshooting |
