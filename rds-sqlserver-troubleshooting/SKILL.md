---
name: rds-sqlserver-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon RDS for SQL Server
  problems by analyzing instance configurations, performance metrics,
  connectivity, parameter groups, and following structured runbooks.
  Activate when: launch failures, instance class issues, storage
  problems, high CPU or SQL Server wait statistics, memory pressure,
  I/O bottlenecks, tempdb contention, connection failures, Windows
  Authentication/Kerberos issues, SSL/TLS problems, parameter group
  issues, SQL Server configuration, max memory/workers tuning,
  native backup to S3, snapshot restore, point-in-time recovery,
  read replica issues, Multi-AZ failover, native backup restore
  migration, DMS issues, collation problems, TDE encryption,
  SQL Server audit, login/user issues, version upgrades, patching,
  or the user says something is wrong with RDS SQL Server without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with RDS, CloudWatch, Performance
  Insights, KMS, CloudTrail, and optionally DMS permissions. SQL Server
  access via SSMS, sqlcmd, or compatible client for database-level
  diagnostics using T-SQL.
---

# RDS SQL Server Diagnostics

## When to use

Any RDS SQL Server investigation where the console alone is insufficient — instance launch failures, performance degradation, SQL Server wait statistics, connectivity issues, Windows Authentication/Kerberos, parameter tuning, backup/recovery, replication, migration, encryption, or upgrade troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rds describe-db-instances --db-instance-identifier <instance-id>
aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440
aws rds describe-db-log-files --db-instance-identifier <instance-id>
aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG
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

### Step 3 — SQL Server-specific diagnostics (via T-SQL)

```sql
-- Active requests and wait types
SELECT r.session_id, r.status, r.command, r.wait_type, r.wait_time,
       r.cpu_time, r.total_elapsed_time, r.reads, r.writes,
       t.text AS sql_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50;

-- Cumulative wait statistics
SELECT TOP 20 wait_type, wait_time_ms/1000.0 AS wait_sec,
       signal_wait_time_ms/1000.0 AS signal_wait_sec,
       waiting_tasks_count
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN ('SLEEP_TASK','BROKER_TO_FLUSH','SQLTRACE_BUFFER_FLUSH',
  'CLR_AUTO_EVENT','CLR_MANUAL_EVENT','LAZYWRITER_SLEEP','CHECKPOINT_QUEUE',
  'WAITFOR','XE_TIMER_EVENT','FT_IFTS_SCHEDULER_IDLE_WAIT','BROKER_TASK_STOP',
  'DIRTY_PAGE_POLL','HADR_FILESTREAM_IOMGR_IOCOMPLETION','SP_SERVER_DIAGNOSTICS_SLEEP')
ORDER BY wait_time_ms DESC;

-- Top SQL by CPU
SELECT TOP 10 qs.total_worker_time/1000 AS total_cpu_ms,
       qs.execution_count, qs.total_elapsed_time/1000 AS total_elapsed_ms,
       qs.total_logical_reads, qs.total_physical_reads,
       SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
         ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
           ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY qs.total_worker_time DESC;

-- Database file space usage
SELECT DB_NAME(database_id) AS db_name, name, type_desc,
       size*8/1024 AS size_mb, FILEPROPERTY(name,'SpaceUsed')*8/1024 AS used_mb
FROM sys.master_files WHERE database_id > 4;
```

Read `references/rds-sqlserver-guardrails.md` before concluding on any RDS SQL Server issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-instances` | Instance config, engine version, storage, Multi-AZ |
| `describe-events` | Recent RDS events and notifications |
| `describe-db-parameters` | Parameter group settings |
| `describe-option-groups` | SQL Server option group features (TDE, audit, etc.) |
| `download-db-log-file-portion` | SQL Server error log (ERRORLOG) |
| `describe-db-snapshots` | Backup and snapshot status |
| `describe-db-log-files` | List available log files |
| `CloudWatch CPUUtilization` | CPU usage and saturation |
| `CloudWatch FreeableMemory` | Available memory (buffer pool pressure) |
| `CloudWatch ReadIOPS/WriteIOPS` | I/O throughput |
| `CloudWatch ReadLatency/WriteLatency` | I/O latency |
| `CloudWatch DatabaseConnections` | Connection count |
| `Performance Insights` | DB load, wait types, top SQL |
| `sys.dm_exec_requests` | Active queries and wait types |
| `sys.dm_os_wait_stats` | Cumulative wait statistics |
| `sys.dm_exec_query_stats` | Query performance statistics |
| `sys.dm_os_memory_clerks` | Memory allocation by component |

## Gotchas: RDS SQL Server

- SQL Server editions on RDS: Express (10 GB limit, limited features), Web (web-only licensing), Standard (most features), Enterprise (all features). License model is License Included for all editions except Enterprise which also supports BYOL.
- No sysadmin role. The RDS master user gets `processadmin`, `setupadmin`, `db_owner` on user databases, and limited `msdb` access. Do not suggest granting sysadmin.
- No Windows Authentication by default. To enable Windows Auth, configure Kerberos with AWS Managed Microsoft AD via AWS Directory Service. Requires domain joining the RDS instance via option group.
- SQL Server Agent is available but limited. No PowerShell job steps, no OS command (CmdExec) steps, no ActiveX steps. Agent jobs run T-SQL and SSIS package steps only. Multi-server administration (MSX/TSX) is not supported.
- No SSMS direct OS access. No RDP, no xp_cmdshell (disabled by default, can be enabled but limited), no access to Windows Event Viewer, no direct file system access.
- MSDB limitations. The master user has limited access to msdb. Some system stored procedures are restricted. Use RDS-specific stored procedures in msdb for native backup/restore.
- TDE (Transparent Data Encryption) requires Enterprise Edition on RDS. Uses AWS KMS for key management. Configured via option group. Not available on Express, Web, or Standard editions.
- Multi-AZ uses Always On Availability Groups under the hood (SQL Server 2012+). You cannot configure AG directly. Failover is automatic and managed by RDS. Typical failover time is 1-2 minutes. No Failover Cluster Instances (FCI) on RDS.
- SQL Server native backup/restore to S3 via msdb stored procedures: `msdb.dbo.rds_backup_database`, `msdb.dbo.rds_restore_database`, `msdb.dbo.rds_task_status`. Requires S3 integration via option group and IAM role.
- Collation is set at instance creation and is immutable. Default is `SQL_Latin1_General_CP1_CI_AS`. Cannot be changed after creation. Database-level collation can differ from instance collation but instance (server) collation is fixed.
- Max server memory is auto-managed by RDS based on instance class. Do not set it manually via parameter group — RDS calculates the optimal value. Over-riding can cause instability.
- Tempdb configuration: RDS manages tempdb file count and sizing. On SQL Server 2016+, RDS creates multiple tempdb data files based on vCPU count (up to 8). You cannot add tempdb files manually.
- Linked servers require option group configuration. Not available by default. Limited to SQL Server linked servers (no Oracle, MySQL linked servers via OLE DB providers on RDS).
- SSIS, SSRS, and SSAS are NOT available on RDS SQL Server. For SSIS, use AWS Glue or run SSIS on EC2. For SSRS, use EC2 or QuickSight. For SSAS, use EC2.
- CLR integration has limited support. Only SAFE assemblies are allowed by default. EXTERNAL_ACCESS and UNSAFE assemblies require additional configuration and are restricted.
- SQL Server audit to S3: configure via option group. Audit files are written to S3 bucket. Server-level and database-level audit specifications are supported.
- Database Mail: configure via option group (SQL Server Mail profile). Not available by default.
- No Failover Cluster Instances (FCI). Multi-AZ uses Always On AG, not Windows Server Failover Clustering.

### SQL Server edition comparison on RDS

| Feature | Express | Web | Standard | Enterprise |
|---------|---------|-----|----------|------------|
| Max DB size | 10 GB | No limit | No limit | No limit |
| Max memory used | 1 GB buffer pool | No limit | 128 GB buffer pool | OS max |
| TDE | No | No | No | Yes |
| Read replicas | No | No | No | Yes |
| Always On AG (Multi-AZ) | No | Yes | Yes | Yes |
| SQL Server Agent | No | Yes | Yes | Yes |
| Compression | No | No | Yes | Yes |
| Partitioning | No | No | No | Yes |
| Column store | No | No | Yes (limited) | Yes |

## Anti-hallucination rules

1. Always cite specific AWS CLI output, CloudWatch metrics, Performance Insights data, or T-SQL DMV query results as evidence.
2. sysadmin role is NOT available on RDS SQL Server. Never suggest granting sysadmin or running commands that require sysadmin. The master user has processadmin and setupadmin roles.
3. SSRS, SSIS, and SSAS are NOT available on RDS. Never suggest installing or configuring these services on RDS SQL Server.
4. Collation cannot be changed after instance creation. Never suggest ALTER SERVER COLLATION or reinstalling SQL Server to change collation.
5. Multi-AZ Always On AG is managed by RDS. Never suggest manual AG configuration, manual failover commands, or direct AG DMV modifications.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1-A3 | Launch failures, instance class issues, storage issues |
| B — Performance | B1-B4 | High CPU/waits, memory pressure, I/O bottleneck, tempdb contention |
| C — Connectivity | C1-C3 | Connection failures, Windows Auth/Kerberos, SSL/TLS |
| D — Parameters | D1-D3 | Parameter group issues, SQL Server config, max memory/workers |
| E — Backup & Recovery | E1-E3 | Native backup to S3, snapshot restore, point-in-time recovery |
| F — Replication | F1-F2 | Read replica issues, Multi-AZ failover |
| G — Migration | G1-G3 | Native backup restore, DMS issues, collation problems |
| H — Security | H1-H3 | TDE issues, SQL Server audit, login/user issues |
| I — Maintenance | I1-I2 | Version upgrades, patching |
| Z — Catch-All | Z1 | General troubleshooting |
