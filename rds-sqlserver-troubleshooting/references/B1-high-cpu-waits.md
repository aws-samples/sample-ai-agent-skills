---
title: "B1 — High CPU and Wait Statistics"
description: "Diagnose high CPU utilization and SQL Server wait statistics on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "high CPU"
  - "CPUUtilization"
  - "slow queries"
  - "wait stats"
  - "db load"
  - "performance degradation"
  - "SOS_SCHEDULER_YIELD"
  - "CXPACKET"
owner: devops-agent
objective: "Identify the root cause of high CPU and correlate with SQL Server wait statistics"
context: "High CPU on RDS SQL Server is typically caused by inefficient queries, missing indexes, excessive compilation/recompilation, or parallelism issues. Performance Insights provides DB load breakdown by wait types. sys.dm_os_wait_stats and sys.dm_exec_requests are the primary diagnostic DMVs."
---

## Phase 1 — Triage

MUST:
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check DB load via Performance Insights:
  ```
  aws pi get-resource-metrics --service-type RDS \
    --identifier db-<resource-id> \
    --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.wait_event"}}]' \
    --start-time <start> --end-time <end> --period-in-seconds 300
  ```
- Identify active requests and wait types:
  ```sql
  SELECT r.session_id, r.status, r.command, r.wait_type, r.wait_time,
         r.cpu_time, r.total_elapsed_time, r.reads, r.writes, r.logical_reads,
         t.text AS sql_text, p.query_plan
  FROM sys.dm_exec_requests r
  CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
  CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
  WHERE r.session_id > 50 AND r.status = 'running'
  ORDER BY r.cpu_time DESC;
  ```
- Identify top SQL by CPU:
  ```sql
  SELECT TOP 10 qs.total_worker_time/1000 AS total_cpu_ms,
         qs.execution_count,
         qs.total_worker_time/qs.execution_count/1000 AS avg_cpu_ms,
         qs.total_logical_reads, qs.total_physical_reads,
         SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
           ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
             ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
  FROM sys.dm_exec_query_stats qs
  CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
  ORDER BY qs.total_worker_time DESC;
  ```

SHOULD:
- Check cumulative wait statistics:
  ```sql
  SELECT TOP 20 wait_type, wait_time_ms/1000.0 AS wait_sec,
         signal_wait_time_ms/1000.0 AS signal_wait_sec,
         waiting_tasks_count,
         wait_time_ms*100.0/SUM(wait_time_ms) OVER() AS pct
  FROM sys.dm_os_wait_stats
  WHERE wait_type NOT IN ('SLEEP_TASK','BROKER_TO_FLUSH','SQLTRACE_BUFFER_FLUSH',
    'CLR_AUTO_EVENT','CLR_MANUAL_EVENT','LAZYWRITER_SLEEP','CHECKPOINT_QUEUE',
    'WAITFOR','XE_TIMER_EVENT','FT_IFTS_SCHEDULER_IDLE_WAIT','BROKER_TASK_STOP',
    'DIRTY_PAGE_POLL','HADR_FILESTREAM_IOMGR_IOCOMPLETION','SP_SERVER_DIAGNOSTICS_SLEEP')
  ORDER BY wait_time_ms DESC;
  ```
- Check for missing indexes:
  ```sql
  SELECT TOP 10 d.statement AS table_name,
         d.equality_columns, d.inequality_columns, d.included_columns,
         s.user_seeks, s.user_scans, s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans) AS improvement_measure
  FROM sys.dm_db_missing_index_details d
  JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
  JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
  ORDER BY improvement_measure DESC;
  ```
- Check for excessive recompilation:
  ```sql
  SELECT TOP 10 plan_generation_num, execution_count,
         qs.total_worker_time/1000 AS total_cpu_ms,
         SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS query_text
  FROM sys.dm_exec_query_stats qs
  CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
  WHERE plan_generation_num > 1
  ORDER BY plan_generation_num DESC;
  ```

MAY:
- Check for parallelism issues (CXPACKET waits): review `cost threshold for parallelism` and `max degree of parallelism` settings
- Check query store for plan regressions (SQL Server 2016+):
  ```sql
  SELECT TOP 10 q.query_id, qt.query_sql_text, rs.avg_cpu_time
  FROM sys.query_store_query q
  JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
  JOIN sys.query_store_plan p ON q.query_id = p.query_id
  JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
  ORDER BY rs.avg_cpu_time DESC;
  ```

## Phase 2 — Remediate

MUST:
- For inefficient SQL: add missing indexes, update statistics
  ```sql
  UPDATE STATISTICS [schema].[table] WITH FULLSCAN;
  ```
- For excessive recompilation: use parameterized queries, add OPTIMIZE FOR hints
- For CPU-bound queries: optimize execution plans, reduce logical reads

SHOULD:
- Tune parallelism settings via parameter group:
  - `cost threshold for parallelism`: increase from default 5 to 25-50
  - `max degree of parallelism`: set to number of cores per NUMA node or 8, whichever is less
- Scale instance class if CPU is legitimately saturated by workload
- Implement connection pooling to reduce session overhead

MAY:
- Use Query Store to force known-good execution plans
- Implement Resource Governor to limit CPU per workload group

## Common Issues

- symptoms: "CPU at 100%, top wait is SOS_SCHEDULER_YIELD"
  diagnosis: "CPU-bound workload. Queries consuming all available CPU."
  resolution: "Optimize top CPU-consuming queries. Add indexes. Scale instance class."

- symptoms: "High CPU with CXPACKET waits"
  diagnosis: "Excessive parallelism. Queries going parallel unnecessarily."
  resolution: "Increase cost threshold for parallelism. Set appropriate MAXDOP."

- symptoms: "High CPU with excessive compilations"
  diagnosis: "Ad-hoc queries causing constant plan compilation."
  resolution: "Use parameterized queries. Enable optimize for ad hoc workloads."

## Safety Ratings
- GREEN: CloudWatch CPUUtilization metrics, Performance Insights, SELECT from sys.dm_exec_requests/sys.dm_os_wait_stats/sys.dm_exec_query_stats — read-only inspection
- YELLOW: UPDATE STATISTICS, CREATE INDEX, modify-db-parameter-group (MAXDOP, cost threshold) — recoverable but may impact performance
- RED: delete-db-instance, force-failover, KILL session — high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "CPU sustained above 90% affecting application response times"
- "Fix requires parameter group change that needs reboot"
- "Fix requires creating indexes on large production tables"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from sys.dm_exec_requests (may contain sensitive data)
- HIGH: query results from sys.dm_exec_requests (contain active SQL text and execution plans)
- MEDIUM: CloudWatch CPU metrics, wait statistics, missing index recommendations

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix CPU issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest killing sessions without confirming with the application team"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If a new index causes write performance degradation, drop it and reassess"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "high_cpu — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<CPUUtilization metrics>"
  - type: wait_stats
    content: "<top wait types from sys.dm_os_wait_stats>"
  - type: top_sql
    content: "<top SQL by CPU from sys.dm_exec_query_stats>"
severity: HIGH
mitigation:
  immediate: "Optimize or kill the offending queries"
  long_term: "Implement query monitoring and index strategy"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
