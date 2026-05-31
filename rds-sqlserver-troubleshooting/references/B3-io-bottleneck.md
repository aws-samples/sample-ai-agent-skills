---
title: "B3 — I/O Bottleneck"
description: "Diagnose I/O performance issues on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "I/O"
  - "IOPS"
  - "latency"
  - "ReadLatency"
  - "WriteLatency"
  - "PAGEIOLATCH"
  - "WRITELOG"
  - "slow disk"
  - "IO_COMPLETION"
owner: devops-agent
objective: "Identify and resolve I/O bottlenecks affecting RDS SQL Server performance"
context: "RDS SQL Server uses EBS storage. I/O performance depends on storage type (gp2/gp3/io1/io2), volume size, and instance class EBS bandwidth. SQL Server I/O waits (PAGEIOLATCH_SH, PAGEIOLATCH_EX, WRITELOG) are the primary indicators."
---

## Phase 1 — Triage

MUST:
- Check I/O metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DiskQueueDepth ...
  ```
- Check storage configuration: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Type:StorageType,Size:AllocatedStorage,IOPS:Iops}'`
- Check I/O wait types:
  ```sql
  SELECT wait_type, wait_time_ms/1000.0 AS wait_sec,
         waiting_tasks_count,
         wait_time_ms/NULLIF(waiting_tasks_count,0) AS avg_wait_ms
  FROM sys.dm_os_wait_stats
  WHERE wait_type IN ('PAGEIOLATCH_SH','PAGEIOLATCH_EX','PAGEIOLATCH_UP',
    'WRITELOG','IO_COMPLETION','ASYNC_IO_COMPLETION')
  ORDER BY wait_time_ms DESC;
  ```
- Check file-level I/O statistics:
  ```sql
  SELECT DB_NAME(vfs.database_id) AS db_name, mf.name, mf.type_desc,
         vfs.num_of_reads, vfs.num_of_writes,
         vfs.io_stall_read_ms/NULLIF(vfs.num_of_reads,0) AS avg_read_ms,
         vfs.io_stall_write_ms/NULLIF(vfs.num_of_writes,0) AS avg_write_ms,
         vfs.io_stall_read_ms + vfs.io_stall_write_ms AS total_io_stall_ms
  FROM sys.dm_io_virtual_file_stats(NULL, NULL) vfs
  JOIN sys.master_files mf ON vfs.database_id = mf.database_id AND vfs.file_id = mf.file_id
  ORDER BY total_io_stall_ms DESC;
  ```

SHOULD:
- Check top I/O consuming queries:
  ```sql
  SELECT TOP 10 qs.total_physical_reads, qs.total_logical_reads,
         qs.execution_count,
         qs.total_physical_reads/NULLIF(qs.execution_count,0) AS reads_per_exec,
         SUBSTRING(st.text, (qs.statement_start_offset/2)+1, 200) AS query_text
  FROM sys.dm_exec_query_stats qs
  CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
  ORDER BY qs.total_physical_reads DESC;
  ```
- Check for table scans causing excessive reads:
  ```sql
  SELECT TOP 10 DB_NAME(s.database_id) AS db_name,
         OBJECT_NAME(s.object_id, s.database_id) AS table_name,
         s.user_scans, s.user_seeks, s.user_lookups
  FROM sys.dm_db_index_usage_stats s
  WHERE s.user_scans > 1000 AND s.database_id > 4
  ORDER BY s.user_scans DESC;
  ```

MAY:
- Check Enhanced Monitoring for OS-level I/O stats
- Review pending I/O requests: `SELECT * FROM sys.dm_io_pending_io_requests;`

## Phase 2 — Remediate

MUST:
- For IOPS limit: upgrade storage type (gp2→gp3 or io1/io2) or increase provisioned IOPS
- For high read latency: optimize SQL to reduce physical reads, add indexes
- For WRITELOG waits: check commit frequency, batch transactions where possible

SHOULD:
- Add missing indexes to reduce table scans
- Update statistics for better query plans: `EXEC sp_updatestats;`
- Consider gp3 for independent IOPS/throughput configuration

MAY:
- Enable read-ahead for sequential scan workloads
- Use columnstore indexes for analytical queries (Standard/Enterprise)

## Common Issues

- symptoms: "High PAGEIOLATCH_SH waits"
  diagnosis: "Data pages being read from disk. Buffer pool miss. Undersized buffer pool or missing indexes."
  resolution: "Add indexes. Scale to larger instance class for more buffer pool. Increase IOPS."

- symptoms: "High WRITELOG waits"
  diagnosis: "Transaction log writes slow. Each COMMIT waits for log flush."
  resolution: "Batch transactions. Reduce commit frequency. Upgrade to io1/io2 for lower write latency."

- symptoms: "DiskQueueDepth consistently > 1"
  diagnosis: "I/O requests queuing. Storage throughput saturated."
  resolution: "Increase provisioned IOPS or switch to higher-performance storage type."

## Safety Ratings
- GREEN: CloudWatch ReadIOPS/WriteIOPS/ReadLatency/WriteLatency/DiskQueueDepth metrics, SELECT from sys.dm_os_wait_stats/sys.dm_io_virtual_file_stats — read-only inspection
- YELLOW: modify-db-instance (storage type/IOPS change), sp_updatestats — recoverable changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "I/O latency causing application timeouts"
- "Fix requires storage type change (6-hour cooldown)"
- "Fix requires parameter group change that needs reboot"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from top I/O queries
- HIGH: query results from sys.dm_exec_requests (contain active SQL text)
- MEDIUM: CloudWatch I/O metrics, storage configuration, file-level I/O statistics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix I/O issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"

## Phase 3 — Rollback
- "Restore from snapshot if storage configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "Wait 6 hours between storage modifications (cooldown period)"

## Output Format

```yaml
root_cause: "io_bottleneck — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<IOPS, latency, queue depth metrics>"
  - type: wait_stats
    content: "<I/O wait types from sys.dm_os_wait_stats>"
  - type: file_io
    content: "<file-level I/O stats from sys.dm_io_virtual_file_stats>"
severity: HIGH
mitigation:
  immediate: "Optimize top I/O queries or increase IOPS"
  long_term: "Right-size storage type and implement I/O monitoring"
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
