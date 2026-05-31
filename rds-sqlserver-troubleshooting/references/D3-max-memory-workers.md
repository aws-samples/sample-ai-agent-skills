---
title: "D3 — Max Memory and Worker Threads"
description: "Diagnose max server memory and max worker threads issues on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "max server memory"
  - "max worker threads"
  - "worker thread exhaustion"
  - "THREADPOOL"
  - "memory limit"
  - "non-yielding scheduler"
owner: devops-agent
objective: "Resolve max server memory and worker thread configuration issues on RDS SQL Server"
context: "RDS auto-manages max server memory based on instance class. Worker threads are calculated automatically based on CPU count. THREADPOOL waits indicate worker thread exhaustion. Overriding max server memory can cause OS-level memory pressure or underutilization."
---

## Phase 1 — Triage

MUST:
- Check max server memory setting:
  ```sql
  SELECT name, value_in_use FROM sys.configurations
  WHERE name IN ('max server memory (MB)','min server memory (MB)');
  ```
- Check actual memory usage:
  ```sql
  SELECT physical_memory_in_use_kb/1024 AS physical_memory_mb,
         locked_page_allocations_kb/1024 AS locked_pages_mb,
         total_virtual_address_space_kb/1024 AS virtual_address_mb,
         process_physical_memory_low, process_virtual_memory_low
  FROM sys.dm_os_process_memory;
  ```
- Check worker thread usage:
  ```sql
  SELECT max_workers_count FROM sys.dm_os_sys_info;
  SELECT scheduler_id, current_tasks_count, runnable_tasks_count,
         current_workers_count, active_workers_count, work_queue_count
  FROM sys.dm_os_schedulers WHERE status = 'VISIBLE ONLINE';
  ```
- Check for THREADPOOL waits:
  ```sql
  SELECT wait_type, wait_time_ms, waiting_tasks_count
  FROM sys.dm_os_wait_stats WHERE wait_type = 'THREADPOOL';
  ```

SHOULD:
- Check FreeableMemory: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check memory by clerk type:
  ```sql
  SELECT TOP 10 type, SUM(pages_kb)/1024 AS size_mb
  FROM sys.dm_os_memory_clerks GROUP BY type ORDER BY SUM(pages_kb) DESC;
  ```
- Check for non-yielding scheduler errors in error log:
  ```
  aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG
  ```

MAY:
- Check parameter group for memory overrides: `aws rds describe-db-parameters --db-parameter-group-name <param-group> --query "Parameters[?ParameterName=='max server memory (MB)']"`
- Review Enhanced Monitoring for OS-level memory

## Phase 2 — Remediate

MUST:
- For max server memory: let RDS manage it (do not override unless specifically needed)
- For THREADPOOL waits: reduce connection count, implement connection pooling
- For worker thread exhaustion: scale to larger instance class (more CPUs = more workers)

SHOULD:
- If max server memory was manually overridden, reset to RDS default:
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=max server memory (MB),ParameterValue={DBInstanceClassMemory/1048576*3/4},ApplyMethod=immediate"
  ```
- Implement connection pooling to reduce worker thread demand
- Monitor worker thread usage with CloudWatch custom metrics

MAY:
- Adjust max worker threads only if recommended by AWS Support
- Use Resource Governor to limit concurrent requests per workload group

## Common Issues

- symptoms: "THREADPOOL waits — new connections timing out"
  diagnosis: "All worker threads in use. Too many concurrent connections or long-running queries."
  resolution: "Kill long-running queries. Implement connection pooling. Scale instance class."

- symptoms: "FreeableMemory near zero after manual max server memory increase"
  diagnosis: "Max server memory set too high, leaving insufficient memory for OS."
  resolution: "Reset max server memory to RDS default. Let RDS manage it."

- symptoms: "Non-yielding scheduler detected"
  diagnosis: "Worker thread stuck. Often caused by I/O issues, memory pressure, or bugs."
  resolution: "Check I/O latency and memory pressure. May require reboot. Open AWS Support case if persistent."

## Safety Ratings
- GREEN: SELECT from sys.configurations/sys.dm_os_process_memory/sys.dm_os_schedulers/sys.dm_os_wait_stats/sys.dm_os_memory_clerks, CloudWatch FreeableMemory — read-only inspection
- YELLOW: modify-db-parameter-group (max server memory), KILL session — recoverable but may impact performance
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "THREADPOOL waits causing connection timeouts"
- "Fix requires parameter group change that needs reboot"
- "Non-yielding scheduler detected — may require reboot"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, memory configuration
- HIGH: error log contents (may contain sensitive error details)
- MEDIUM: memory clerk details, worker thread statistics, CloudWatch metrics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix memory/worker issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest overriding max server memory without understanding RDS memory management"
- "NEVER suggest modifying max worker threads without AWS Support guidance"

## Phase 3 — Rollback
- "Reset max server memory to RDS default if manual override causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "max_memory_workers — <specific_cause>"
evidence:
  - type: memory_config
    content: "<max server memory setting and actual usage>"
  - type: worker_threads
    content: "<worker thread count and THREADPOOL waits>"
severity: HIGH
mitigation:
  immediate: "Reset memory to RDS default or kill long-running queries"
  long_term: "Let RDS manage memory, implement connection pooling"
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
