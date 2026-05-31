---
name: rds-mysql-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon RDS for MySQL
  problems by analyzing instance configurations, performance metrics,
  connectivity, parameter groups, and following structured runbooks.
  Activate when: launch failures, slow queries, InnoDB buffer pool
  issues, lock waits, connectivity failures, parameter group tuning,
  read replica lag, Multi-AZ failover, backup/PITR failures, storage
  IOPS or autoscaling issues, RDS Proxy problems, encryption config,
  version upgrades, or the user says something is wrong with RDS MySQL.
compatibility: >
  Requires AWS CLI or SDK access with RDS, CloudWatch, Performance
  Insights, KMS, CloudTrail permissions. MySQL client access for
  database-level diagnostics.
---

# RDS MySQL Diagnostics

## When to use

Any RDS MySQL investigation where the console alone is insufficient — instance launch failures, slow queries, InnoDB buffer pool pressure, lock waits/deadlocks, connectivity issues, parameter group tuning, replication lag, Multi-AZ failover, backup/recovery, storage performance, RDS Proxy, encryption, or upgrade troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rds describe-db-instances --db-instance-identifier <instance-id>
aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
aws rds describe-db-log-files --db-instance-identifier <instance-id>
aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/mysql-error-running.log
aws rds describe-db-parameters --db-parameter-group-name <param-group>
```

### Step 2 — Performance deep dive

```
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
  --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace ...
aws pi get-resource-metrics --service-type RDS \
  --identifier db-<resource-id> \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time <start> --end-time <end> --period-in-seconds 300
```

### Step 3 — MySQL-specific diagnostics (via SQL)

```sql
-- Active threads and lock waits
SELECT * FROM information_schema.PROCESSLIST WHERE COMMAND != 'Sleep' ORDER BY TIME DESC;
SELECT * FROM information_schema.INNODB_TRX;
SELECT * FROM performance_schema.data_lock_waits;

-- InnoDB buffer pool hit ratio
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
SELECT (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_ratio
FROM (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_reads FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') a,
     (SELECT VARIABLE_VALUE AS Innodb_buffer_pool_read_requests FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests') b;

-- Slow query analysis
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 20;
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- Replication status (on read replica)
SHOW REPLICA STATUS\G
```

Read `references/guardrails.md` before concluding on any RDS MySQL issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-instances` | Instance config, engine version, storage, Multi-AZ |
| `describe-events` | Recent RDS events and notifications |
| `describe-db-parameters` | Parameter group settings |
| `download-db-log-file-portion` | Error log, slow query log |
| `describe-db-snapshots` | Backup and snapshot status |
| `CloudWatch CPUUtilization` | CPU usage |
| `CloudWatch FreeableMemory` | Available memory (buffer pool pressure) |
| `CloudWatch ReadIOPS/WriteIOPS` | I/O throughput |
| `CloudWatch ReadLatency/WriteLatency` | I/O latency |
| `CloudWatch DatabaseConnections` | Connection count |
| `Performance Insights` | DB load, wait events, top SQL |
| `SHOW PROCESSLIST` | Active queries and lock waits |
| `INNODB_TRX` | Active InnoDB transactions |
| `SHOW REPLICA STATUS` | Replication lag and errors |

## Gotchas: RDS MySQL

- InnoDB is the only supported storage engine for production. MyISAM tables are not crash-safe and not replicated reliably. RDS defaults to InnoDB.
- `innodb_buffer_pool_size` defaults to 75% of instance memory via `{DBInstanceClassMemory*3/4}`. Over-allocating causes OOM kills.
- No `SUPER` privilege on RDS MySQL. Use `rds_superuser_role` or stored procedures like `mysql.rds_kill`, `mysql.rds_set_configuration`.
- Read replicas use MySQL native async replication. Lag is expected under write-heavy workloads. Monitor `ReplicaLag` CloudWatch metric.
- Multi-AZ uses synchronous replication to a standby (not a read replica). Failover is automatic, typically 60-120 seconds. DNS endpoint stays the same.
- RDS Proxy pools connections at the proxy level. Application connection strings must point to the proxy endpoint, not the DB endpoint.
- Storage autoscaling: set `--max-allocated-storage` to enable. Scaling happens when free space < 10% and lasts > 5 minutes. Cooldown is 6 hours.
- Binary logging (`binlog_format`) must be ROW for replication. MIXED or STATEMENT can cause replication inconsistencies.
- `max_connections` is derived from instance memory: `{DBInstanceClassMemory/12582880}`. Override carefully — too many connections cause OOM.
- PITR restores to a new instance. You cannot restore in-place. The new instance gets a new endpoint.
- Major version upgrades (e.g., 5.7→8.0) require pre-upgrade checks. Use `mysql-shell` upgrade checker or RDS pre-upgrade validation.

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, Performance Insights data, or MySQL query results as evidence.
2. No SUPER privilege on RDS. Never suggest `SET GLOBAL` for restricted variables or `CHANGE MASTER TO` directly.
3. MyISAM is not recommended on RDS. Never suggest MyISAM for production tables.
4. PITR creates a new instance. Never suggest in-place point-in-time restore.
5. Multi-AZ standby is not readable. Never suggest querying the Multi-AZ standby directly.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## Runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1 | Launch failures |
| B — Performance | B1-B3 | Slow queries, InnoDB buffer pool, lock waits |
| C — Connectivity | C1 | Connection failures |
| D — Parameters | D1 | Parameter group issues |
| E — Replication | E1-E2 | Read replicas, Multi-AZ failover |
| F — Backup | F1 | Backup and PITR |
| G — Storage | G1-G2 | IOPS performance, storage autoscaling |
| H — Upgrades | H1 | Version upgrades |
| I — Proxy | I1 | RDS Proxy |
| J — Security | J1 | Encryption |
| Z — Catch-All | Z1 | General troubleshooting |
