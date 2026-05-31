---
title: "B2 — Memory Pressure"
description: "Diagnose SQL Server memory pressure on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "memory pressure"
  - "FreeableMemory"
  - "buffer pool"
  - "plan cache"
  - "RESOURCE_SEMAPHORE"
  - "out of memory"
  - "page life expectancy"
  - "lazy writes"
owner: devops-agent
objective: "Identify and resolve SQL Server memory allocation issues on RDS"
context: "RDS manages max server memory based on instance class. SQL Server memory includes buffer pool (data cache), plan cache, and memory grants for query execution. Memory pressure manifests as low page life expectancy, high lazy writes, RESOURCE_SEMAPHORE waits, or low FreeableMemory in CloudWatch."
---

## Phase 1 — Triage

MUST:
- Check FreeableMemory: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check memory clerks:
  ```sql
  SELECT TOP 15 type AS memory_clerk, SUM(pages_kb)/1024 AS size_mb
  FROM sys.dm_os_memory_clerks
  GROUP BY type ORDER BY SUM(pages_kb) DESC;
  ```
- Check buffer pool usage:
  ```sql
  SELECT DB_NAME(database_id) AS db_name,
         COUNT(*)*8/1024 AS buffer_pool_mb
  FROM sys.dm_os_buffer_descriptors
  GROUP BY database_id ORDER BY buffer_pool_mb DESC;
  ```
- Check page life expectancy:
  ```sql
  SELECT object_name, counter_name, cntr_value
  FROM sys.dm_os_performance_counters
  WHERE counter_name IN ('Page life expectancy','Buffer cache hit ratio',
    'Lazy writes/sec','Page reads/sec','Free pages','Target pages');
  ```

SHOULD:
- Check for RESOURCE_SEMAPHORE waits (memory grant waits):
  ```sql
  SELECT session_id, wait_type, wait_time, resource_description
  FROM sys.dm_os_waiting_tasks
  WHERE wait_type = 'RESOURCE_SEMAPHORE';
  ```
- Check memory grants:
  ```sql
  SELECT session_id, requested_memory_kb/1024 AS requested_mb,
         granted_memory_kb/1024 AS granted_mb,
         used_memory_kb/1024 AS used_mb,
         ideal_memory_kb/1024 AS ideal_mb
  FROM sys.dm_exec_query_memory_grants
  WHERE granted_memory_kb IS NOT NULL
  ORDER BY granted_memory_kb DESC;
  ```
- Check plan cache bloat:
  ```sql
  SELECT objtype, COUNT(*) AS plan_count,
         SUM(size_in_bytes)/1024/1024 AS size_mb,
         SUM(usecounts) AS total_uses
  FROM sys.dm_exec_cached_plans
  GROUP BY objtype ORDER BY size_mb DESC;
  ```

MAY:
- Check for memory-optimized tables (In-Memory OLTP) consuming memory
- Review Enhanced Monitoring for OS-level memory breakdown

## Phase 2 — Remediate

MUST:
- For plan cache bloat: enable optimize for ad hoc workloads
  ```sql
  -- Check current setting
  SELECT name, value_in_use FROM sys.configurations WHERE name = 'optimize for ad hoc workloads';
  ```
  Set via parameter group: `optimize for ad hoc workloads = 1`
- For persistent memory pressure: scale to a larger instance class with more memory
- For RESOURCE_SEMAPHORE: identify and optimize queries requesting large memory grants

SHOULD:
- Clear plan cache as emergency measure (causes brief performance dip):
  ```sql
  DBCC FREEPROCCACHE;
  ```
- Optimize queries with large memory grants (sorts, hash joins on large datasets)
- Update statistics to improve cardinality estimates and memory grant accuracy:
  ```sql
  EXEC sp_updatestats;
  ```

MAY:
- Use Resource Governor to limit memory grants per workload group
- Implement connection pooling to reduce total memory consumption

## Common Issues

- symptoms: "RESOURCE_SEMAPHORE waits, queries waiting for memory grants"
  diagnosis: "Queries requesting more memory than available. Often caused by bad cardinality estimates."
  resolution: "Update statistics. Optimize queries with large sorts/hash joins. Scale instance class."

- symptoms: "Page life expectancy below 300 seconds"
  diagnosis: "Buffer pool under pressure. Data pages being evicted too quickly."
  resolution: "Scale to larger instance class. Optimize queries to reduce logical reads."

- symptoms: "Plan cache consuming excessive memory"
  diagnosis: "Ad-hoc query plans filling plan cache. Each unique query string gets its own plan."
  resolution: "Enable optimize for ad hoc workloads. Use parameterized queries."

## Safety Ratings
- GREEN: CloudWatch FreeableMemory metrics, SELECT from sys.dm_os_memory_clerks/sys.dm_os_buffer_descriptors/sys.dm_os_performance_counters — read-only inspection
- YELLOW: modify-db-parameter-group (optimize for ad hoc workloads), DBCC FREEPROCCACHE, sp_updatestats — recoverable but may cause brief performance dip
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "RESOURCE_SEMAPHORE waits causing query timeouts"
- "Fix requires parameter group change that needs reboot"
- "FreeableMemory critically low"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, memory clerk details
- HIGH: query results from sys.dm_exec_query_memory_grants (contain SQL text and memory allocation)
- MEDIUM: CloudWatch memory metrics, buffer pool statistics, plan cache sizes

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix memory issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest reducing max server memory below SQL Server minimum requirements"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If DBCC FREEPROCCACHE causes performance degradation, it will recover as plan cache warms up"

## Output Format

```yaml
root_cause: "memory_pressure — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<FreeableMemory metrics>"
  - type: memory_clerks
    content: "<sys.dm_os_memory_clerks output>"
  - type: buffer_pool
    content: "<buffer pool usage by database>"
severity: HIGH
mitigation:
  immediate: "Free plan cache or kill memory-heavy queries"
  long_term: "Right-size instance class and optimize query memory grants"
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
