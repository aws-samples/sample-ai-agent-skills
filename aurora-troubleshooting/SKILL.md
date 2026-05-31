---
name: aurora-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Aurora (MySQL
  and PostgreSQL) problems by analyzing cluster configurations,
  performance metrics, connectivity, parameter groups, and following
  structured runbooks. Activate when: cluster creation failures,
  storage issues, cluster endpoint confusion, cluster parameter groups,
  high CPU or wait events, memory pressure, Aurora I/O model issues,
  query performance, connection failures, endpoint confusion, RDS Proxy
  issues, connection limits, writer failover, reader failover, DNS
  propagation, reader lag, Aurora MySQL binlog replication, Aurora
  PostgreSQL logical replication, Serverless v2 scaling issues, cold
  start, capacity allocation, Global Database replication lag, planned
  failover, unplanned failover, backup failures, backtrack (MySQL only),
  clone issues, encryption, IAM database authentication, DMS to Aurora,
  MySQL/PostgreSQL migration, or the user says something is wrong with
  Aurora without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with RDS, CloudWatch, Performance
  Insights, KMS, CloudTrail, and optionally DMS permissions. MySQL
  client or psql for engine-specific database-level diagnostics.
---

# Aurora Diagnostics

## When to use

Any Aurora investigation where the console alone is insufficient — cluster creation failures, storage issues, performance degradation, Aurora-specific wait events, connectivity issues, endpoint confusion, failover events, replication lag, Serverless v2 scaling, Global Database issues, backup/recovery, encryption, or migration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rds describe-db-clusters --db-cluster-identifier <cluster-id>
aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>
aws rds describe-events --source-identifier <cluster-id> --source-type db-cluster --duration 1440
aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <cluster-param-group>
aws rds describe-db-parameters --db-parameter-group-name <instance-param-group>
aws rds describe-db-log-files --db-instance-identifier <instance-id>
aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/mysql-error-running.log
```

### Step 2 — Performance deep dive

```
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
  --dimensions Name=DBClusterIdentifier,Value=<cluster-id> \
  --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory \
  --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeBytesUsed ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeReadIOPs ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeWriteIOPs ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraReplicaLag ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ServerlessDatabaseCapacity ...
aws pi get-resource-metrics --service-type RDS \
  --identifier db-<resource-id> \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time <start> --end-time <end> --period-in-seconds 300
```

### Step 3 — Engine-specific diagnostics

#### Aurora MySQL

```sql
-- Active threads and wait events
SELECT thread_id, processlist_user, processlist_host, processlist_command,
       processlist_state, processlist_time, processlist_info
FROM performance_schema.threads
WHERE processlist_command != 'Sleep' AND type = 'FOREGROUND';

-- Top waits
SELECT event_name, count_star, sum_timer_wait/1000000000 AS total_wait_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE count_star > 0 ORDER BY sum_timer_wait DESC LIMIT 20;

-- InnoDB status
SHOW ENGINE INNODB STATUS\G

-- Replication lag (on reader)
SHOW SLAVE STATUS\G

-- Connection usage
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';
```

#### Aurora PostgreSQL

```sql
-- Active queries and wait events
SELECT pid, usename, client_addr, state, wait_event_type, wait_event,
       query_start, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active' AND pid != pg_backend_pid();

-- Top wait events
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE state = 'active' AND wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event ORDER BY count DESC;

-- Table bloat and vacuum stats
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum, last_analyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Replication lag (on reader)
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Connection usage
SELECT count(*) AS total_connections FROM pg_stat_activity;
SHOW max_connections;
```

Read `references/aurora-guardrails.md` before concluding on any Aurora issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-clusters` | Cluster config, engine, endpoints, storage, Global Database |
| `describe-db-instances` (filtered) | Instance details within a cluster |
| `describe-events` (db-cluster) | Recent cluster events and notifications |
| `describe-db-cluster-parameters` | Cluster-level parameter group settings |
| `describe-db-parameters` | Instance-level parameter group settings |
| `download-db-log-file-portion` | MySQL error log or PostgreSQL log |
| `describe-db-cluster-snapshots` | Cluster snapshot status |
| `CloudWatch CPUUtilization` | CPU usage per instance |
| `CloudWatch FreeableMemory` | Available memory per instance |
| `CloudWatch VolumeBytesUsed` | Aurora storage volume usage |
| `CloudWatch VolumeReadIOPs/VolumeWriteIOPs` | Aurora I/O operations |
| `CloudWatch AuroraReplicaLag` | Reader replica lag |
| `CloudWatch ServerlessDatabaseCapacity` | Serverless v2 ACU usage |
| `CloudWatch AuroraBinlogReplicaLag` | Binlog replication lag (MySQL) |
| `Performance Insights` | DB load, wait events, top SQL |
| `performance_schema` (MySQL) | Wait events, thread activity |
| `pg_stat_activity` (PostgreSQL) | Active queries, wait events |

## Gotchas: Aurora

- Aurora architecture: shared distributed storage volume with 6 copies of data across 3 AZs. Storage is separate from compute. One writer instance, up to 15 reader instances. Storage auto-scales up to 128 TiB but cannot shrink — it only grows.
- Aurora MySQL vs Aurora PostgreSQL: different engines with different feature sets. Backtrack is MySQL only. Parallel query is MySQL only. Logical replication setup differs. Aurora MySQL uses binlog replication for external targets; Aurora PostgreSQL uses native logical replication.
- Aurora Serverless v2: scales between min and max ACU (Aurora Capacity Units). 0.5 ACU increments. Min ACU can be as low as 0.5. Scaling is not instant — there is latency during scale-up. Cold start can occur when scaling from very low ACU. Each ACU provides ~2 GiB of memory.
- Reader endpoint: uses DNS round-robin to distribute connections across reader instances. It is NOT a load balancer — it does not balance at the connection level or query level. New DNS lookups may resolve to the same reader. For true load balancing, use RDS Proxy or application-level routing.
- Writer failover: typical failover time is ~30 seconds. DNS TTL for the cluster endpoint is 5 seconds. Applications must handle DNS caching and reconnection. If a reader exists, it is promoted to writer. If no reader exists, a new writer is created (slower).
- Aurora Global Database: RPO < 1 second, RTO < 1 minute for managed planned failover. Unplanned failover (detach and promote) has higher RPO. Managed planned failover maintains replication topology. Unplanned failover requires manual replication re-setup.
- Aurora cloning: uses copy-on-write protocol. Clone creation is near-instant regardless of database size. The clone shares storage pages with the source until pages are modified. Clone is a full independent cluster — not a snapshot.
- Aurora backtrack (MySQL only): rewinds the cluster to a specific point in time without creating a new cluster. Backtrack window is configurable (up to 72 hours). Backtrack affects the entire cluster (all readers). Not available on Aurora PostgreSQL.
- Storage auto-scaling: grows automatically in 10 GiB increments up to 128 TiB. Storage never shrinks even if data is deleted — the space is reused internally. To reclaim storage, you must create a new cluster from a snapshot.
- Aurora I/O-Optimized vs Standard: I/O-Optimized eliminates I/O charges but has higher instance cost (~30% more). Best for I/O-heavy workloads. Can switch between configurations once every 30 days.
- Performance Insights: free tier retains 7 days of data. Long-term retention (up to 2 years) requires paid tier. Available for both Aurora MySQL and Aurora PostgreSQL.
- Parallel query (MySQL only): pushes query processing down to the Aurora storage layer. Best for analytical queries on large tables. Not available on Aurora PostgreSQL. Requires specific instance classes and engine versions.
- Aurora ML integration: allows calling SageMaker and Comprehend from SQL queries. Available for both Aurora MySQL (via stored functions) and Aurora PostgreSQL (via aws_ml extension).
- Local storage for temp tables: each Aurora instance has local NVMe storage for temporary tables and sort operations. This storage is instance-specific and not shared across the cluster. Size depends on instance class.
- Connection management: RDS Proxy is recommended for Aurora to handle connection pooling, failover routing, and IAM authentication. Proxy supports both MySQL and PostgreSQL protocols.
- Aurora MySQL binary log replication: used for replicating to external MySQL targets or between Aurora clusters. Binlog must be explicitly enabled (binlog_format parameter). Enabling binlog adds write latency.
- Aurora PostgreSQL logical replication: uses native PostgreSQL logical replication (publication/subscription). Requires setting `rds.logical_replication` to 1 and reboot. WAL level changes to logical.
- Cluster parameter groups vs instance parameter groups: Aurora has two levels. Cluster parameter groups apply to all instances in the cluster (engine-level settings). Instance parameter groups apply to individual instances (instance-specific tuning). Some parameters exist only at one level.
- Aurora blue/green deployments: creates a staging environment (green) that is a copy of production (blue). Switchover promotes green to production with minimal downtime. Useful for major version upgrades and parameter changes.

### Aurora MySQL vs Aurora PostgreSQL comparison

| Feature | Aurora MySQL | Aurora PostgreSQL |
|---------|-------------|-------------------|
| Backtrack | Yes (up to 72h) | No |
| Parallel Query | Yes | No |
| Binlog Replication | Yes | N/A (use logical replication) |
| Logical Replication | Limited | Yes (native pub/sub) |
| Blue/Green Deployments | Yes | Yes |
| Global Database | Yes | Yes |
| Serverless v2 | Yes | Yes |
| RDS Proxy | Yes | Yes |
| Performance Insights | Yes | Yes |
| IAM DB Authentication | Yes | Yes |

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, Performance Insights data, or engine-specific SQL query results as evidence.
2. Aurora storage never shrinks. Never suggest that deleting data will free up billed storage. Storage is reused internally but the volume size does not decrease.
3. Reader endpoint is DNS round-robin, NOT a load balancer. Never suggest it provides connection-level or query-level load balancing.
4. Backtrack is Aurora MySQL only. Never suggest backtrack for Aurora PostgreSQL clusters.
5. Aurora Global Database managed planned failover and unplanned failover (detach and promote) are different operations with different RPO/RTO characteristics. Do not conflate them.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 34 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A4 | Cluster creation failures, storage issues, cluster endpoint issues, cluster parameter groups |
| B — Performance | B1-B4 | High CPU/waits, memory pressure, I/O issues (Aurora I/O model), query performance |
| C — Connectivity | C1-C4 | Connection failures, endpoint confusion, RDS Proxy issues, connection limits |
| D — Failover | D1-D3 | Writer failover, reader failover, DNS propagation |
| E — Replication | E1-E3 | Reader lag, Aurora MySQL binlog replication, Aurora PostgreSQL logical replication |
| F — Serverless | F1-F3 | Scaling issues, cold start, capacity allocation |
| G — Global Database | G1-G3 | Replication lag, planned failover, unplanned failover |
| H — Backup & Recovery | H1-H3 | Backup failures, backtrack (MySQL), clone issues |
| I — Security | I1-I2 | Encryption, IAM database authentication |
| J — Migration | J1-J2 | DMS to Aurora, MySQL/PostgreSQL migration |
| Z — Catch-All | Z1 | General troubleshooting |
