---
title: "B2 — Memory Pressure"
description: "Diagnose memory pressure on Aurora instances"
status: active
severity: HIGH
triggers:
  - "FreeableMemory"
  - "out of memory"
  - "OOM"
  - "memory pressure"
  - "swap usage"
  - "buffer pool"
  - "shared_buffers"
owner: devops-agent
objective: "Identify and resolve memory pressure issues on Aurora instances"
context: "Aurora instances have memory allocated based on instance class. Aurora MySQL uses InnoDB buffer pool; Aurora PostgreSQL uses shared_buffers and work_mem. Local storage (NVMe) is used for temp tables. Memory pressure leads to increased I/O, swap usage, and potential OOM kills."
---

## Phase 1 — Triage

MUST:
- Check FreeableMemory:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Average Minimum
  ```
- Check SwapUsage:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name SwapUsage \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  ```
- For Aurora MySQL — check buffer pool usage:
  ```sql
  SHOW STATUS LIKE 'Innodb_buffer_pool_pages%';
  SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
  SELECT * FROM sys.memory_global_total;
  ```
- For Aurora PostgreSQL — check memory usage:
  ```sql
  SHOW shared_buffers;
  SHOW work_mem;
  SHOW maintenance_work_mem;
  SELECT count(*) AS connections FROM pg_stat_activity;
  ```

SHOULD:
- Check connection count (each connection consumes memory):
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  ```
- For Aurora MySQL — check memory consumers:
  ```sql
  SELECT event_name, current_alloc FROM sys.memory_global_by_current_bytes LIMIT 20;
  ```
- For Aurora PostgreSQL — check for memory-intensive queries:
  ```sql
  SELECT pid, usename, query, state FROM pg_stat_activity
  WHERE state = 'active' ORDER BY backend_start;
  ```
- Check Enhanced Monitoring for OS-level memory breakdown

MAY:
- Check for memory leaks (steadily decreasing FreeableMemory over days)
- Review temp table usage on local storage

## Phase 2 — Remediate

MUST:
- If FreeableMemory is critically low: scale up instance class for more memory
- Reduce connection count: implement connection pooling (RDS Proxy recommended)
- For Aurora MySQL: tune `innodb_buffer_pool_size` if using custom parameter group
- For Aurora PostgreSQL: tune `shared_buffers`, `work_mem`, `maintenance_work_mem`

SHOULD:
- Optimize queries that use large temp tables or sort operations
- Implement connection pooling to reduce per-connection memory overhead
- Set appropriate `max_connections` based on instance class

MAY:
- Enable Enhanced Monitoring for detailed memory breakdown
- Consider Serverless v2 for workloads with variable memory needs

## Common Issues

- symptoms: "FreeableMemory dropping to near zero"
  diagnosis: "Too many connections, oversized buffer pool, or memory-intensive queries."
  resolution: "Reduce connections (use RDS Proxy). Scale up instance. Optimize queries."

- symptoms: "SwapUsage increasing"
  diagnosis: "Instance running out of physical memory, OS swapping to disk."
  resolution: "Scale up instance class. Reduce memory consumers. Lower max_connections."

- symptoms: "OOM killer terminating processes (PostgreSQL)"
  diagnosis: "work_mem * active queries exceeds available memory."
  resolution: "Lower work_mem. Reduce concurrent queries. Scale up instance."

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch FreeableMemory/SwapUsage/DatabaseConnections metrics, SHOW STATUS/VARIABLES, pg_stat_activity — read-only inspection
- YELLOW: modify-db-instance (scale up), modify-db-parameter-group (tune buffer pool/shared_buffers) — recoverable but may require reboot
- RED: force-failover, delete-db-instance — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "FreeableMemory critically low with OOM risk"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, memory configuration values
- HIGH: query results from sys.memory_global_by_current_bytes, pg_stat_activity (contain internal memory allocation details and SQL text)
- MEDIUM: CloudWatch memory metrics, connection counts, buffer pool statistics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix memory issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest reducing innodb_buffer_pool_size or shared_buffers drastically without testing impact"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes (innodb_buffer_pool_size, shared_buffers, work_mem) and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "If instance scale-up causes issues, scale back down after confirming application compatibility"

## Output Format

```yaml
root_cause: "memory_pressure — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<FreeableMemory, SwapUsage metrics>"
  - type: memory_config
    content: "<buffer pool or shared_buffers settings>"
  - type: connections
    content: "<connection count>"
severity: HIGH
mitigation:
  immediate: "Scale up instance or reduce memory consumers"
  long_term: "Implement connection pooling and memory monitoring"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
