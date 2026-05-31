---
name: rds-oracle-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon RDS for Oracle
  problems by analyzing instance configurations, performance metrics,
  connectivity, parameter groups, and following structured runbooks.
  Activate when: launch failures, instance class issues, storage
  problems, high CPU or Oracle wait events, SGA/PGA memory pressure,
  I/O bottlenecks, connection failures, TNS errors, listener issues,
  parameter group problems, Oracle init parameter tuning, session/process
  limits, backup failures, snapshot restore, point-in-time recovery,
  read replica issues, Multi-AZ failover, Data Pump migration, DMS
  issues, character set problems, TDE encryption, SSL/native network
  encryption, Oracle audit, version upgrades, patching, or the user
  says something is wrong with RDS Oracle without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with RDS, CloudWatch, Performance
  Insights, KMS, CloudTrail, and optionally DMS permissions. Oracle
  SQL access via SQL*Plus or compatible client for database-level
  diagnostics.
---

# RDS Oracle Diagnostics

## When to use

Any RDS Oracle investigation where the console alone is insufficient — instance launch failures, performance degradation, Oracle wait events, connectivity issues, TNS errors, parameter tuning, backup/recovery, replication, migration, encryption, or upgrade troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rds describe-db-instances --db-instance-identifier <instance-id>
aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
aws rds describe-db-log-files --db-instance-identifier <instance-id>
aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name trace/alert_<SID>.log
aws rds describe-db-parameters --db-parameter-group-name <param-group>
aws rds describe-option-groups --option-group-name <option-group>
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
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace ...
aws pi get-resource-metrics --service-type RDS \
  --identifier db-<resource-id> \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  --start-time <start> --end-time <end> --period-in-seconds 300
```

### Step 3 — Oracle-specific diagnostics (via SQL)

```sql
-- Active sessions and wait events
SELECT sid, serial#, username, status, event, wait_class, seconds_in_wait
FROM v$session WHERE status = 'ACTIVE' AND username IS NOT NULL;

-- SGA/PGA memory usage
SELECT component, current_size/1024/1024 AS mb FROM v$sga_dynamic_components;
SELECT name, value/1024/1024 AS mb FROM v$pgastat WHERE name IN ('total PGA allocated','maximum PGA allocated');

-- Top SQL by elapsed time
SELECT sql_id, elapsed_time/1000000 AS elapsed_sec, executions, buffer_gets
FROM v$sql ORDER BY elapsed_time DESC FETCH FIRST 10 ROWS ONLY;

-- Tablespace usage
SELECT tablespace_name, ROUND(used_percent,2) AS pct_used FROM dba_tablespace_usage_metrics;
```

Read `references/rds-oracle-guardrails.md` before concluding on any RDS Oracle issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-instances` | Instance config, engine version, storage, Multi-AZ |
| `describe-events` | Recent RDS events and notifications |
| `describe-db-parameters` | Parameter group settings |
| `describe-option-groups` | Oracle option group features (TDE, SSL, etc.) |
| `download-db-log-file-portion` | Alert log, trace files, listener log |
| `describe-db-snapshots` | Backup and snapshot status |
| `describe-db-log-files` | List available log files |
| `CloudWatch CPUUtilization` | CPU usage and saturation |
| `CloudWatch FreeableMemory` | Available memory (SGA/PGA pressure) |
| `CloudWatch ReadIOPS/WriteIOPS` | I/O throughput |
| `CloudWatch ReadLatency/WriteLatency` | I/O latency |
| `CloudWatch DatabaseConnections` | Connection count |
| `Performance Insights` | DB load, wait events, top SQL |
| `v$session` | Active sessions and wait events |
| `v$sga_dynamic_components` | SGA memory allocation |
| `v$pgastat` | PGA memory statistics |
| `v$sql` | SQL performance statistics |

## Gotchas: RDS Oracle

- Oracle licensing on RDS: License Included (SE2 only) or BYOL (SE2 and EE). BYOL requires existing Oracle licenses with Software Update License & Support. Verify license model before launch.
- Oracle-specific parameter groups: key parameters include `processes`, `sessions`, `sga_target`, `pga_aggregate_target`, `open_cursors`, `db_files`. RDS manages some parameters automatically — not all init.ora parameters are modifiable.
- No RAC on RDS. Oracle Real Application Clusters is not supported. For high availability, use Multi-AZ (which uses Oracle Data Guard under the hood).
- No customer-managed Data Guard. Multi-AZ uses Data Guard internally but you cannot configure standby databases, switchover, or failover manually. RDS manages it.
- No ASM (Automatic Storage Management). RDS uses EBS volumes exclusively. No raw devices, no ASM disk groups.
- Character set (AL32UTF8 default) cannot be changed after instance creation. Plan character set requirements before launch. National character set (AL16UTF16) is also immutable.
- Oracle editions: SE2 (Standard Edition Two) supports up to 16 vCPUs. EE (Enterprise Edition) required for partitioning, Advanced Compression, Advanced Security (TDE), Active Data Guard (read replicas).
- Oracle storage: EBS only (gp2, gp3, io1, io2). No raw devices. Storage autoscaling available. Maximum 64 TiB.
- Oracle Statspack available on all editions. AWR (Automatic Workload Repository) requires EE with Diagnostics Pack license. Use Statspack on SE2.
- Oracle Data Pump (expdp/impdp) for migration: use the `DBMS_DATAPUMP` PL/SQL package via the RDS-provided wrapper procedures in the `rdsadmin` schema. No OS-level access to run expdp/impdp directly.
- Oracle-specific CloudWatch metrics: `CPUUtilization`, `FreeableMemory`, `ReadIOPS`, `WriteIOPS`, `ReadLatency`, `WriteLatency`, `DatabaseConnections`, `FreeStorageSpace`, `SwapUsage`. Enhanced Monitoring provides OS-level metrics.
- Oracle SSL/TLS: two options — Oracle Native Network Encryption (via option group, no certificates needed) or SSL (requires certificates, option group configuration). Native encryption is simpler but SSL provides certificate-based authentication.
- Oracle TDE (Transparent Data Encryption): requires EE with Advanced Security option. Uses AWS KMS or CloudHSM for key management. Configured via option group. Tablespace-level or column-level encryption.
- Oracle backup: automated backups use storage-level snapshots, not RMAN. You cannot access RMAN directly. For logical backups, use Data Pump via `rdsadmin` procedures.
- Oracle version upgrade paths: not all versions can upgrade directly. Check `describe-db-engine-versions --engine oracle-ee --engine-version <current>` for valid upgrade targets. Major version upgrades may require parameter group changes.
- Multi-AZ uses Oracle Data Guard (physical standby) under the hood. Failover is automatic. Typical failover time is 1-2 minutes. DNS endpoint stays the same.
- Read replicas use Active Data Guard and require Enterprise Edition. Read replicas are available in same region or cross-region. They use the same endpoint pattern but with a different identifier.

### Oracle edition comparison

| Feature | SE2 | EE |
|---------|-----|----|
| Max vCPUs | 16 | No limit |
| Partitioning | No | Yes |
| Advanced Compression | No | Yes |
| TDE | No | Yes |
| Active Data Guard (read replicas) | No | Yes |
| Diagnostics Pack (AWR) | No | Yes |
| Tuning Pack | No | Yes |
| OLAP | No | Yes |
| Label Security | No | Yes |

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, Performance Insights data, or Oracle SQL query results as evidence.
2. RAC is NOT available on RDS. Never suggest RAC configuration or RAC-specific troubleshooting.
3. RMAN is NOT directly accessible on RDS. Never suggest running RMAN commands. Use automated backups and Data Pump via `rdsadmin`.
4. Character set cannot be changed after creation. Never suggest ALTER DATABASE CHARACTER SET on RDS.
5. Multi-AZ Data Guard is managed by RDS. Never suggest manual Data Guard configuration, switchover, or failover commands.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1-A3 | Launch failures, instance class issues, storage issues |
| B — Performance | B1-B4 | High CPU/wait events, memory pressure (SGA/PGA), I/O bottleneck, Oracle-specific waits |
| C — Connectivity | C1-C3 | Connection failures, TNS issues, listener issues |
| D — Parameters | D1-D3 | Parameter group issues, Oracle init parameters, session/process limits |
| E — Backup & Recovery | E1-E3 | Automated backup failures, snapshot restore, point-in-time recovery |
| F — Replication | F1-F2 | Read replica issues, Multi-AZ failover |
| G — Migration | G1-G3 | Data Pump issues, DMS issues, character set problems |
| H — Security | H1-H3 | TDE issues, SSL/native encryption, Oracle audit |
| I — Maintenance | I1-I2 | Version upgrades, patching |
| Z — Catch-All | Z1 | General troubleshooting |
