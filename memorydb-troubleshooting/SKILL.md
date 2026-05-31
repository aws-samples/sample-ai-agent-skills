---
name: memorydb-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon MemoryDB for Redis
  problems by analyzing cluster configurations, node health, shard
  distribution, CloudWatch metrics, and following structured runbooks.
  Activate when: cluster creation failures, node issues, shard configuration
  problems, latency spikes, throughput degradation, memory pressure,
  connection failures, ACL authentication errors, transaction log issues,
  data persistence questions, vertical scaling, horizontal scaling
  (resharding), snapshot failures, restore issues, ACL management,
  encryption configuration, or the user says something is wrong with
  MemoryDB without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MemoryDB, CloudWatch, CloudTrail,
  EC2, KMS, and optionally redis-cli for direct node diagnostics.
---

# MemoryDB Diagnostics

## When to use

Any Amazon MemoryDB investigation where the console alone is insufficient — cluster health analysis, performance debugging, connectivity troubleshooting, durability verification, scaling operations, backup/restore, security configuration, or comparison with ElastiCache.

## Investigation workflow

### Step 1 — Collect and triage

```
aws memorydb describe-clusters
aws memorydb describe-clusters --cluster-name <cluster-name>
aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name CPUUtilization --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name DatabaseMemoryUsagePercentage --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name EngineCPUUtilization --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name CurrConnections --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws memorydb describe-events --source-type cluster --duration 1440
aws memorydb describe-events --source-type node --duration 1440
aws memorydb describe-parameter-groups
aws memorydb describe-parameters --parameter-group-name <param-group>
aws memorydb describe-acls
aws memorydb describe-users
redis-cli -h <cluster-endpoint> -p 6379 --tls INFO
redis-cli -h <cluster-endpoint> -p 6379 --tls SLOWLOG GET 25
redis-cli -h <cluster-endpoint> -p 6379 --tls CLUSTER INFO
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=memorydb.amazonaws.com --max-results 20
aws memorydb describe-snapshots --cluster-name <cluster-name>
aws memorydb describe-subnet-groups --subnet-group-name <subnet-group>
aws ec2 describe-security-groups --group-ids <sg-id>
aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name ReplicationLag --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 60 --statistics Maximum
```

Read `references/memorydb-guardrails.md` before concluding on any MemoryDB issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-clusters` | Cluster configuration, shards, and node details |
| `describe-events` | Recent cluster and node events |
| `describe-parameter-groups` | Parameter group settings |
| `describe-acls` | Access control list configuration |
| `describe-users` | User authentication details |
| `describe-snapshots` | Snapshot/backup details |
| `describe-subnet-groups` | Subnet group and VPC configuration |
| `redis-cli INFO` | Real-time server statistics |
| `redis-cli SLOWLOG` | Slow command analysis |
| `redis-cli CLUSTER INFO` | Cluster topology and health |

## Gotchas: Amazon MemoryDB

- MemoryDB is durable (multi-AZ transaction log) unlike ElastiCache (cache only). MemoryDB writes data to a durable multi-AZ transaction log before acknowledging writes. This provides data durability across restarts and failures. ElastiCache is a cache — data may be lost on node failure. This durability adds write latency compared to ElastiCache.
- ACL is the only authentication method. MemoryDB uses Access Control Lists (ACLs) with users for authentication. There is no AUTH token option like ElastiCache. Each user has a username and password. ACLs define which commands and keys users can access. The default user (open-access) should be restricted in production.
- Data persists across restarts. Unlike ElastiCache where data may be lost, MemoryDB data survives node restarts, failovers, and even full cluster restarts because of the transaction log. This is a key differentiator from ElastiCache.
- Redis compatibility but not all commands supported. MemoryDB is compatible with Redis but does not support all Redis commands. Commands like DEBUG, MIGRATE, CLUSTER FAILOVER (manual), and some administrative commands are restricted. Always check MemoryDB documentation for supported commands.
- Cluster mode is always enabled. MemoryDB always operates in cluster mode — there is no non-cluster mode option. Data is distributed across shards using hash slots (0-16383). Applications must use Redis Cluster-aware clients. Single-shard clusters are possible but still use cluster mode protocol.
- Pricing is higher than ElastiCache due to durability. MemoryDB costs more per node than equivalent ElastiCache nodes because of the transaction log infrastructure. Choose MemoryDB when durability is required. Use ElastiCache when data can be regenerated from a primary data store.
- Snapshot from MemoryDB can restore to ElastiCache but not vice versa. MemoryDB snapshots can be used to create ElastiCache clusters, enabling migration from MemoryDB to ElastiCache. However, ElastiCache snapshots cannot be used to create MemoryDB clusters. Migration from ElastiCache to MemoryDB requires a different approach.
- TLS is always enabled. MemoryDB requires TLS for all connections — it cannot be disabled. All clients must support TLS. Use redis-cli with --tls flag. This differs from ElastiCache where TLS is optional.

## Anti-hallucination rules

1. Always cite specific cluster names, node IDs, CloudWatch metrics, or redis-cli output as evidence.
2. MemoryDB is NOT ElastiCache. Never confuse durability guarantees, authentication methods, or pricing between the two.
3. ACL is the only auth method. Never suggest AUTH token authentication for MemoryDB.
4. Cluster mode is always enabled. Never suggest non-cluster mode configuration.
5. TLS is always required. Never suggest connecting without TLS.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A3 | Creation failures, node issues, shard configuration |
| B — Performance | B1-B3 | Latency, throughput, memory pressure |
| C — Connectivity | C1-C2 | Connection failures, ACL authentication |
| D — Durability | D1-D2 | Transaction log, data persistence vs ElastiCache |
| E — Scaling | E1-E2 | Vertical scaling, horizontal scaling (resharding) |
| F — Backup | F1-F2 | Snapshot failures, restore issues |
| G — Security | G1-G2 | ACL management, encryption |
| Z — Catch-All | Z1 | General troubleshooting |
