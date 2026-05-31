---
title: "B3 — pg_stat Performance Analysis"
description: "Use pg_stat views for deep performance analysis on RDS PostgreSQL"
status: active
severity: HIGH
triggers:
  - "slow queries"
  - "high CPU"
  - "pg_stat"
  - "performance"
  - "query optimization"
---

## Phase 1 — Triage

MUST:
- Enable pg_stat_statements (requires parameter group change and reboot):
  ```sql
  -- Add to shared_preload_libraries in parameter group, then:
  CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
  ```
- Top queries by total time:
  ```sql
  SELECT queryid, calls, total_exec_time/1000 AS total_sec, mean_exec_time AS mean_ms,
         rows, shared_blks_hit, shared_blks_read, query
  FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
  ```
- Check cache hit ratio:
  ```sql
  SELECT sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_ratio
  FROM pg_statio_user_tables;
  -- Target: > 0.95
  ```
- Check index usage:
  ```sql
  SELECT schemaname, relname, seq_scan, idx_scan,
         ROUND(seq_scan::numeric / NULLIF(seq_scan + idx_scan, 0) * 100, 2) AS seq_scan_pct
  FROM pg_stat_user_tables WHERE seq_scan > 100 ORDER BY seq_scan DESC LIMIT 20;
  ```

SHOULD:
- Check for missing indexes:
  ```sql
  SELECT relname, seq_scan, seq_tup_read, idx_scan
  FROM pg_stat_user_tables WHERE seq_scan > 1000 AND idx_scan = 0 ORDER BY seq_tup_read DESC;
  ```
- Check wait events via Performance Insights
- Review `EXPLAIN (ANALYZE, BUFFERS)` for slow queries

## Phase 2 — Remediate

- Add indexes for high seq_scan tables
- Optimize queries with high mean_exec_time
- Increase shared_buffers if cache hit ratio < 95% (via parameter group)
- Reset stats after optimization: `SELECT pg_stat_statements_reset();`
- Use `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` to identify bottlenecks

## Safety Ratings
- GREEN: SELECT from pg_stat_statements/pg_stat_user_tables/pg_stat_activity, EXPLAIN ANALYZE — read-only inspection
- YELLOW: CREATE INDEX, ANALYZE, modify-db-parameter-group — recoverable but may impact performance
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires creating indexes on large production tables"
- "Fix requires parameter group change that needs reboot"
- "CPU sustained above 90% affecting application response times"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from pg_stat_statements (may contain sensitive data)
- HIGH: EXPLAIN ANALYZE output (reveals table structure and data distribution)
- MEDIUM: CloudWatch metrics, table statistics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix performance issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "If a new index causes write performance degradation, drop it and reassess"
- "Revert parameter group changes and reboot if needed"

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
