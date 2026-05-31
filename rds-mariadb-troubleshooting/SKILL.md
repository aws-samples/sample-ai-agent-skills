---
name: rds-mariadb-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon RDS for MariaDB
  problems by analyzing instance configurations, performance metrics,
  connectivity, parameter groups, and following structured runbooks.
  Activate when: launch failures, performance issues, connectivity
  failures, parameter group tuning, replication lag, backup/PITR,
  storage issues, MariaDB-specific features, encryption, upgrades,
  or the user says something is wrong with RDS MariaDB.
compatibility: >
  Requires AWS CLI or SDK access with RDS, CloudWatch, Performance
  Insights, KMS, CloudTrail permissions. MariaDB/MySQL client for
  database-level diagnostics.
---

# RDS MariaDB Diagnostics

## When to use

Any RDS MariaDB investigation where the console alone is insufficient — instance launch failures, slow queries, InnoDB/Aria engine issues, lock waits, connectivity, parameter tuning, replication, backup/recovery, storage performance, MariaDB-specific features, encryption, or upgrade troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rds describe-db-instances --db-instance-identifier <instance-id>
aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/mariadb-error-running.log
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
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
aws pi get-resource-metrics --service-type RDS \
  --identifier db-<resource-id> \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time <start> --end-time <end> --period-in-seconds 300
```

### Step 3 — MariaDB-specific diagnostics (via SQL)

```sql
-- Active threads and lock waits
SELECT * FROM information_schema.PROCESSLIST WHERE COMMAND != 'Sleep' ORDER BY TIME DESC;
SELECT * FROM information_schema.INNODB_TRX;

-- InnoDB buffer pool
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

-- MariaDB thread pool status
SHOW GLOBAL STATUS LIKE 'Threadpool%';

-- Replication status (on replica)
SHOW SLAVE STATUS\G
-- or SHOW REPLICA STATUS\G (MariaDB 10.5+)
```

Read `references/guardrails.md` before concluding on any RDS MariaDB issue.

## Gotchas: RDS MariaDB

- MariaDB on RDS supports the Aria storage engine (crash-safe MyISAM replacement) for internal temporary tables, but InnoDB remains the recommended production engine.
- MariaDB thread pool is available on RDS and enabled by default. It handles high-concurrency workloads better than MySQL's one-thread-per-connection model.
- `innodb_buffer_pool_size` defaults to `{DBInstanceClassMemory*3/4}`. Same memory constraints as MySQL.
- No `SUPER` privilege. Use `mysql.rds_kill`, `mysql.rds_set_configuration` stored procedures.
- MariaDB uses its own replication implementation (GTID differs from MySQL GTID). Do not confuse MariaDB GTID with MySQL GTID.
- MariaDB 10.x versioning: RDS supports specific MariaDB versions. Check `aws rds describe-db-engine-versions --engine mariadb`.
- System-versioned tables (temporal tables) are a MariaDB-specific feature available on RDS.
- Sequence engine is available in MariaDB on RDS for generating sequences.
- PITR creates a new instance. Cannot restore in-place.
- Multi-AZ uses synchronous replication. Standby is not readable.

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, or SQL query results as evidence.
2. No SUPER privilege. Never suggest direct `SET GLOBAL` for restricted variables.
3. MariaDB GTID ≠ MySQL GTID. Do not apply MySQL GTID troubleshooting to MariaDB.
4. PITR creates a new instance. Never suggest in-place restore.
5. Multi-AZ standby is not readable. Never suggest querying the standby.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## Runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1 | Launch failures |
| B — Performance | B1-B2 | Slow queries/performance, lock waits |
| C — Connectivity | C1 | Connection failures |
| D — Parameters | D1 | Parameter group issues |
| E — Replication | E1-E2 | Read replicas, Multi-AZ failover |
| F — Backup | F1 | Backup and PITR |
| G — Storage | G1 | Storage and IOPS |
| H — Upgrades | H1 | Version upgrades |
| I — Features | I1 | MariaDB-specific features |
| J — Security | J1 | Encryption |
| Z — Catch-All | Z1 | General troubleshooting |
