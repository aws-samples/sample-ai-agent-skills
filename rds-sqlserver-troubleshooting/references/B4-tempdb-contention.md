---
title: "B4 — Tempdb Contention"
description: "Diagnose tempdb contention and performance issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "tempdb"
  - "PAGELATCH"
  - "PFS contention"
  - "GAM contention"
  - "SGAM contention"
  - "tempdb full"
  - "version store"
owner: devops-agent
objective: "Identify and resolve tempdb contention and space issues on RDS SQL Server"
context: "RDS manages tempdb file count automatically. On SQL Server 2016+, RDS creates multiple tempdb data files based on vCPU count (up to 8). Tempdb contention manifests as PAGELATCH waits on PFS, GAM, or SGAM pages. Tempdb space issues arise from large sorts, version store bloat, or temp table usage."
---

## Phase 1 — Triage

MUST:
- Check tempdb space usage:
  ```sql
  SELECT SUM(unallocated_extent_page_count)*8/1024 AS free_mb,
         SUM(user_object_reserved_page_count)*8/1024 AS user_objects_mb,
         SUM(internal_object_reserved_page_count)*8/1024 AS internal_objects_mb,
         SUM(version_store_reserved_page_count)*8/1024 AS version_store_mb
  FROM sys.dm_db_file_space_usage;
  ```
- Check tempdb file configuration:
  ```sql
  SELECT file_id, name, type_desc, size*8/1024 AS size_mb,
         growth, is_percent_growth
  FROM sys.master_files WHERE database_id = DB_ID('tempdb');
  ```
- Check for PAGELATCH waits on tempdb:
  ```sql
  SELECT session_id, wait_type, wait_time, resource_description
  FROM sys.dm_os_waiting_tasks
  WHERE wait_type LIKE 'PAGELATCH%'
    AND resource_description LIKE '2:%';  -- database_id 2 = tempdb
  ```
- Check active requests using tempdb:
  ```sql
  SELECT r.session_id, r.command, r.status,
         t.text AS sql_text,
         ssu.user_objects_alloc_page_count*8/1024 AS user_obj_mb,
         ssu.internal_objects_alloc_page_count*8/1024 AS internal_obj_mb
  FROM sys.dm_exec_requests r
  CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
  JOIN sys.dm_db_session_space_usage ssu ON r.session_id = ssu.session_id
  WHERE ssu.user_objects_alloc_page_count + ssu.internal_objects_alloc_page_count > 0
  ORDER BY (ssu.user_objects_alloc_page_count + ssu.internal_objects_alloc_page_count) DESC;
  ```

SHOULD:
- Check version store usage (if RCSI or snapshot isolation enabled):
  ```sql
  SELECT DB_NAME(database_id) AS db_name,
         reserved_page_count*8/1024 AS version_store_mb
  FROM sys.dm_tran_version_store_space_usage;
  ```
- Check for long-running transactions holding version store:
  ```sql
  SELECT session_id, transaction_id, elapsed_time_seconds
  FROM sys.dm_tran_active_snapshot_database_transactions
  ORDER BY elapsed_time_seconds DESC;
  ```

MAY:
- Check tempdb contention via Performance Insights wait events
- Review Enhanced Monitoring for tempdb I/O patterns

## Phase 2 — Remediate

MUST:
- For tempdb space full: identify and terminate sessions consuming excessive tempdb
- For version store bloat: find and resolve long-running snapshot transactions
- For PAGELATCH contention: scale to a larger instance class (more vCPUs = more tempdb files)

SHOULD:
- Optimize queries that create large temp tables or use excessive sorts
- Reduce use of explicit temp tables where table variables or CTEs suffice
- For version store: consider disabling RCSI if not needed

MAY:
- Enable trace flag 1118 (uniform extent allocation) via parameter group for older SQL Server versions
- Monitor tempdb usage with regular DMV queries

## Common Issues

- symptoms: "PAGELATCH_EX waits on 2:1:1 (PFS page in tempdb)"
  diagnosis: "PFS page contention in tempdb. Too many concurrent temp object allocations."
  resolution: "Scale to larger instance class for more tempdb files. RDS auto-creates files based on vCPU count."

- symptoms: "Tempdb full — version store consuming all space"
  diagnosis: "Long-running snapshot transaction preventing version store cleanup."
  resolution: "Find and terminate the long-running transaction. Check for RCSI on databases."

- symptoms: "Tempdb full — large sort or hash operations"
  diagnosis: "Queries spilling to tempdb due to insufficient memory grants."
  resolution: "Optimize queries. Update statistics for better cardinality estimates. Scale instance class."

## Safety Ratings
- GREEN: SELECT from sys.dm_db_file_space_usage/sys.master_files/sys.dm_os_waiting_tasks/sys.dm_db_session_space_usage — read-only inspection
- YELLOW: KILL session (tempdb consumers), modify-db-instance (scale up) — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Tempdb full causing query failures"
- "Fix requires killing active sessions consuming tempdb"
- "Version store bloat from long-running snapshot transactions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from tempdb-consuming sessions
- MEDIUM: tempdb space usage, file configuration, version store statistics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix tempdb issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest killing sessions without confirming with the application team"
- "NEVER suggest disabling RCSI without understanding application isolation requirements"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "If killing sessions causes application errors, investigate query patterns"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "tempdb_contention — <specific_cause>"
evidence:
  - type: tempdb_space
    content: "<tempdb space usage breakdown>"
  - type: wait_stats
    content: "<PAGELATCH wait details>"
  - type: tempdb_files
    content: "<tempdb file configuration>"
severity: MEDIUM
mitigation:
  immediate: "Kill sessions consuming excessive tempdb or resolve long transactions"
  long_term: "Scale instance class for more tempdb files and optimize queries"
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
