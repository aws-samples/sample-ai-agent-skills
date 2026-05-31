---
name: elasticache-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon ElastiCache problems
  (both Redis and Memcached engines) by analyzing cluster configurations,
  replication groups, parameter groups, CloudWatch metrics, slow logs, and
  following structured runbooks.
  Activate when: connection failures, timeout issues, DNS/endpoint resolution,
  high CPU utilization, memory pressure or evictions, high latency, slow
  commands, replication lag, failover issues, read replica problems, vertical
  or horizontal scaling, ElastiCache Serverless issues, eviction storms,
  memory fragmentation, maxmemory configuration, AUTH token issues, encryption
  problems, security group/subnet issues, backup failures, restore issues,
  slot migration, resharding failures, cross-slot errors, Memcached
  auto-discovery, consistent hashing, Global Datastore replication or
  failover, or the user says something is wrong with ElastiCache without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ElastiCache, CloudWatch, CloudTrail,
  EC2 (for security groups/subnets), KMS, and optionally SNS permissions.
  Redis CLI (redis-cli) recommended for direct node diagnostics.
---

# ElastiCache Diagnostics

## When to use

Any ElastiCache investigation where the console alone is insufficient — connection debugging, performance analysis, replication troubleshooting, scaling operations, memory management, security configuration, backup/restore, cluster mode operations, Memcached-specific issues, or Global Datastore problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws elasticache describe-cache-clusters --show-cache-node-info
aws elasticache describe-replication-groups
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name Evictions --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CurrConnections --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws elasticache describe-events --source-type cache-cluster --duration 1440
aws elasticache describe-events --source-type replication-group --duration 1440
aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group>
redis-cli -h <endpoint> -p 6379 INFO
redis-cli -h <endpoint> -p 6379 SLOWLOG GET 25
redis-cli -h <endpoint> -p 6379 INFO replication
redis-cli -h <endpoint> -p 6379 INFO memory
redis-cli -h <endpoint> -p 6379 CLIENT LIST
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticache.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name ReplicationLag --dimensions Name=CacheClusterId,Value=<replica-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum
aws elasticache describe-cache-subnet-groups --cache-subnet-group-name <subnet-group>
aws ec2 describe-security-groups --group-ids <sg-id>
aws elasticache describe-snapshots --replication-group-id <repl-group-id>
aws elasticache describe-global-replication-groups
```

Read `references/elasticache-guardrails.md` before concluding on any ElastiCache issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-cache-clusters` | Full cluster details and node info |
| `describe-replication-groups` | Redis replication topology and endpoints |
| `describe-events` | Recent cluster and replication group events |
| `describe-cache-parameters` | Parameter group settings |
| `describe-cache-subnet-groups` | Subnet group and VPC configuration |
| `describe-snapshots` | Backup/snapshot details (Redis only) |
| `describe-global-replication-groups` | Global Datastore configuration |
| `describe-cache-engine-versions` | Available engine versions |
| `describe-service-updates` | Pending service updates |
| `redis-cli INFO` | Real-time Redis server stats |
| `redis-cli SLOWLOG` | Slow command analysis |
| `redis-cli CLIENT LIST` | Active connection details |

## Gotchas: ElastiCache

- Cluster mode enabled vs disabled (Redis): Cluster mode enabled (CME) distributes data across multiple shards using hash slots (0-16383). Cluster mode disabled (CMD) has a single shard with one primary and up to 5 replicas. CME supports up to 500 nodes. CMD cannot be converted to CME without creating a new cluster. CME requires the Redis Cluster client.
- Replication group vs cache cluster terminology: A replication group is the Redis logical grouping (primary + replicas). Each node in the replication group is a cache cluster. The replication group endpoint is the primary endpoint. Do not confuse cache cluster IDs with replication group IDs.
- Failover behavior (Multi-AZ): Multi-AZ with automatic failover promotes a read replica to primary when the primary fails. DNS propagation takes 30-60 seconds. Applications using the primary endpoint will automatically connect to the new primary. Failover can cause brief write unavailability (typically under 60 seconds).
- Eviction policies: Redis supports allkeys-lru, volatile-lru, allkeys-lfu, volatile-lfu, allkeys-random, volatile-random, volatile-ttl, and noeviction. Default is volatile-lru. volatile-* policies only evict keys with TTL set. If no keys have TTL and policy is volatile-*, Redis returns OOM errors. allkeys-lru is recommended for cache-only workloads.
- maxmemory-reserved: AWS recommends reserving 25% of maxmemory for non-data overhead (replication buffer, Lua scripts, connections, pub/sub). Default is set per node type. Insufficient reservation causes OOM during replication or heavy write loads. This is set in the parameter group, not in Redis config directly.
- Connection limits per node type: Each node type has a maximum connection limit (maxclients). cache.t3.micro supports ~65,000 connections. Larger node types support more. Monitor CurrConnections in CloudWatch. Connection storms can exhaust limits and cause new connection failures.
- Redis AUTH token rotation: AUTH tokens can be rotated using modify-replication-group with --auth-token and --auth-token-update-strategy (SET or ROTATE). ROTATE allows both old and new tokens during transition. SET immediately replaces the token. Use ROTATE for zero-downtime rotation.
- Redis 7.x vs 6.x feature differences: Redis 7.x adds Redis Functions (server-side scripting), ACL improvements, sharded pub/sub, multi-part AOF, and listpack encoding. Redis 6.x introduced ACLs, client-side caching, and RESP3. Not all Redis OSS features are available in ElastiCache — check AWS documentation.
- Memcached auto-discovery: ElastiCache Memcached supports auto-discovery — clients automatically detect nodes added or removed. Requires the ElastiCache Cluster Client (not standard memcached clients). The configuration endpoint returns the current node list. Standard clients must be manually updated when nodes change.
- ElastiCache Serverless vs provisioned: Serverless automatically scales compute and memory. No node type selection needed. Pricing is based on data stored (per GB-hour) and ElastiCache Processing Units (ECPUs). Serverless has different limits than provisioned (e.g., max item size, connection limits). Not all Redis commands are supported in Serverless.
- Encryption in-transit and at-rest: In-transit encryption (TLS) can only be enabled at cluster creation time — cannot be added later. At-rest encryption uses AWS KMS or ElastiCache-managed keys. Enabling TLS adds ~25% CPU overhead. TLS requires redis-cli with --tls flag or stunnel for older clients.
- Subnet group and security group requirements: ElastiCache clusters must be in a VPC. The subnet group defines which subnets (and AZs) the cluster can use. Security groups control inbound access — port 6379 (Redis) or 11211 (Memcached). Clusters are not publicly accessible — use a bastion host or VPN.
- Parameter group changes: Some parameter changes apply immediately, others require a reboot. Changes to maxmemory-policy, timeout, and tcp-keepalive apply immediately. Changes to maxmemory-reserved, cluster-enabled, and appendonly require a reboot. Check the "Apply Type" column in describe-cache-parameters.
- Backup and restore (Redis only, not Memcached): Only Redis supports snapshots/backups. Memcached is purely in-memory with no persistence. Redis backups use fork() which temporarily doubles memory usage. Schedule backups during low-traffic periods. Backup window and retention are configured per replication group.
- Global Datastore for Redis: Enables cross-region replication with <1 second typical replication lag. Supports one primary region and up to two secondary regions. Failover to secondary region is manual (not automatic). Secondary clusters are read-only. Requires Redis 5.0.6+ and cluster mode enabled.
- Redis Streams, pub/sub fan-out memory: Redis Streams (XADD/XREAD) consume memory proportional to stream length. Pub/sub output buffers can grow unbounded if subscribers are slow — configure client-output-buffer-limit. Fan-out to many subscribers multiplies memory usage. Monitor output buffer metrics.
- Cluster mode resharding: Online resharding allows adding/removing shards without downtime. Slot migration moves hash slots between shards. During resharding, multi-key operations on keys in migrating slots may fail with MOVED/ASK errors. Resharding large shards can take hours. Monitor ReplicationLag during resharding.
- Slow log analysis: Redis SLOWLOG records commands exceeding slowlog-log-slower-than (default 10,000 microseconds = 10ms). Use SLOWLOG GET to retrieve entries. Common slow commands: KEYS, SORT, LRANGE on large lists, SMEMBERS on large sets, HGETALL on large hashes. Replace with SCAN-family commands.

## Anti-hallucination rules

1. Always cite specific cluster configurations, parameter group values, CloudWatch metrics, or redis-cli output as evidence.
2. Never confuse replication group IDs with cache cluster IDs. A replication group contains one or more cache clusters (nodes).
3. Memcached does NOT support replication, persistence, backups, or failover. Never suggest these features for Memcached clusters.
4. In-transit encryption (TLS) can only be enabled at cluster creation time. Never suggest enabling TLS on an existing non-TLS cluster without recreation.
5. ElastiCache clusters are NOT publicly accessible. Never suggest connecting directly from the internet — always require VPC access (bastion, VPN, or VPC peering).
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 32 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Connectivity | A1-A3 | Connection failures, timeout issues, DNS/endpoint resolution |
| B — Performance | B1-B4 | High CPU, memory pressure/evictions, high latency, slow commands |
| C — Replication | C1-C3 | Replication lag, failover issues, read replica problems |
| D — Scaling | D1-D3 | Vertical scaling, horizontal scaling (resharding), ElastiCache Serverless |
| E — Memory | E1-E3 | Eviction storms, memory fragmentation, maxmemory configuration |
| F — Security | F1-F3 | AUTH token issues, encryption, security group/subnet |
| G — Backup | G1-G2 | Backup failures, restore issues |
| H — Cluster Mode | H1-H3 | Slot migration, resharding failures, cross-slot errors |
| I — Memcached | I1-I2 | Auto-discovery, consistent hashing |
| J — Global Datastore | J1-J2 | Replication lag, failover |
| Z — Catch-All | Z1 | General troubleshooting |
