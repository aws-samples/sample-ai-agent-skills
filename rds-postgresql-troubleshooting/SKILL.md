---
name: rds-postgresql-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon RDS for PostgreSQL
  problems by analyzing instance configurations, performance metrics,
  connectivity, parameter groups, and following structured runbooks.
  Activate when: launch failures, vacuum/bloat issues, connection pooling,
  pg_stat analysis, connectivity failures, parameter group tuning,
  replication lag, logical replication, backup/PITR, extensions,
  RDS Proxy, encryption, upgrades, or the user says something is wrong
  with RDS PostgreSQL.
compatibility: >
  Requires AWS CLI or SDK access with RDS, CloudWatch, Performance
  Insights, KMS, CloudTrail permissions. PostgreSQL client (psql) for
  database-level diagnostics.
---

# RDS PostgreSQL Diagnostics

## When to use

Any RDS PostgreSQL investigation where the console alone is insufficient — instance launch failures, vacuum/autovacuum issues, table bloat, connection pooling, pg_stat analysis, connectivity, parameter tuning, physical and logical replication, backup/recovery, extensions, RDS Proxy, encryption, or upgrade troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rds describe-db-instances --db-instance-identifier <instance-id>
aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
aws rds describe-db-log-files --db-instance-identifier <instance-id>
aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/postgresql.log
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
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name TransactionLogsDiskUsage ...
aws pi get-resource-metrics --service-type RDS \
  --identifier db-<resource-id> \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time <start> --end-time <end> --period-in-seconds 300
```

### Step 3 — PostgreSQL-specific diagnostics (via psql)

```sql
-- Active queries and wait events
SELECT pid, usename, state, wait_event_type, wait_event, query_start, query
FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start;

-- Table bloat and dead tuples
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup,0) * 100, 2) AS dead_pct,
       last_vacuum, last_autovacuum, last_analyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Connection usage
SELECT count(*) AS total, state FROM pg_stat_activity GROUP BY state;
SHOW max_connections;

-- Replication status (on primary)
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

Read `references/guardrails.md` before concluding on any RDS PostgreSQL issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-instances` | Instance config, engine version, storage |
| `describe-events` | Recent RDS events |
| `describe-db-parameters` | Parameter group settings |
| `download-db-log-file-portion` | PostgreSQL error log |
| `CloudWatch CPUUtilization` | CPU usage |
| `CloudWatch FreeableMemory` | Available memory (shared_buffers pressure) |
| `CloudWatch TransactionLogsDiskUsage` | WAL disk usage |
| `Performance Insights` | DB load, wait events, top SQL |
| `pg_stat_activity` | Active queries and connections |
| `pg_stat_user_tables` | Table stats, vacuum status, dead tuples |
| `pg_stat_replication` | Replication lag |
| `pg_stat_bgwriter` | Checkpoint and bgwriter stats |

## Gotchas: RDS PostgreSQL

- `shared_buffers` defaults to `{DBInstanceClassMemory/32768}` (≈25% of memory). PostgreSQL relies on OS cache for the rest. Do not set shared_buffers > 40% of instance memory.
- No `rds_superuser` is not the same as PostgreSQL `superuser`. Some operations (e.g., `CREATE EXTENSION`, `pg_terminate_backend`) work, but OS-level access and some superuser-only functions are restricted.
- Autovacuum is critical. Disabling or under-tuning autovacuum leads to table bloat, transaction ID wraparound, and performance degradation. Monitor `n_dead_tup` and `last_autovacuum`.
- Transaction ID wraparound: PostgreSQL uses 32-bit transaction IDs. If autovacuum cannot keep up, the database enters read-only mode at 2 billion transactions. Monitor `age(datfrozenxid)`.
- `max_connections` default is `LEAST({DBInstanceClassMemory/9531392}, 5000)`. Each connection uses ~10MB. Use connection pooling (RDS Proxy or PgBouncer) for high-connection workloads.
- Logical replication requires `rds.logical_replication=1` and `wal_level=logical`. This increases WAL generation. Not all data types and DDL are replicated.
- Extensions must be allowlisted by RDS. Use `SHOW rds.allowed_extensions` or check `aws rds describe-db-engine-versions --engine postgres --engine-version <ver> --query 'DBEngineVersions[0].SupportedFeatureNames'`.
- PITR restores to a new instance. WAL-based, so restore time depends on WAL volume since last snapshot.
- Major version upgrades (e.g., 14→16) use pg_upgrade internally. Extensions must be compatible with the target version.

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, Performance Insights data, or PostgreSQL query results as evidence.
2. No OS-level access. Never suggest editing `postgresql.conf` or `pg_hba.conf` directly.
3. `rds_superuser` is not PostgreSQL `superuser`. Some superuser-only functions are restricted.
4. PITR creates a new instance. Never suggest in-place restore.
5. Autovacuum should never be disabled. Never suggest turning off autovacuum globally.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## Runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1 | Launch failures |
| B — Performance | B1-B3 | Vacuum/bloat, connection pooling, pg_stat analysis |
| C — Connectivity | C1 | Connection failures |
| D — Parameters | D1 | Parameter group issues |
| E — Replication | E1-E2 | Physical replication, logical replication |
| F — Backup | F1 | Backup and PITR |
| G — Extensions | G1 | Extension management |
| H — Upgrades | H1 | Version upgrades |
| I — Proxy | I1 | RDS Proxy |
| J — Security | J1 | Encryption |
| Z — Catch-All | Z1 | General troubleshooting |
