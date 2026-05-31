---
title: "B4 — Query Performance"
description: "Diagnose query performance issues on Aurora MySQL and Aurora PostgreSQL"
status: active
severity: HIGH
triggers:
  - "slow query"
  - "query timeout"
  - "execution plan"
  - "missing index"
  - "full table scan"
  - "query performance"
owner: devops-agent
objective: "Identify and resolve query performance issues on Aurora"
context: "Query performance issues on Aurora can stem from missing indexes, stale statistics, suboptimal execution plans, lock contention, or resource constraints. Diagnosis differs between Aurora MySQL (EXPLAIN, performance_schema, slow query log) and Aurora PostgreSQL (EXPLAIN ANALYZE, pg_stat_statements, auto_explain)."
---

## Phase 1 — Triage

MUST:
- Check Performance Insights for top SQL:
  ```
  aws pi get-resource-metrics --service-type RDS --identifier db-<resource-id> \
    --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.sql"}}]' \
    --start-time <start> --end-time <end> --period-in-seconds 300
  ```
- For Aurora MySQL — check slow queries:
  ```sql
  -- Enable slow query log if not enabled
  -- Cluster parameter: slow_query_log = 1, long_query_time = 1

  -- Check recent slow queries
  SELECT digest_text, count_star, avg_timer_wait/1000000000 AS avg_ms,
         sum_rows_examined, sum_rows_sent,
         sum_rows_examined/NULLIF(sum_rows_sent,0) AS examine_to_send_ratio
  FROM performance_schema.events_statements_summary_by_digest
  ORDER BY avg_timer_wait DESC LIMIT 10;

  -- Explain a specific query
  EXPLAIN FORMAT=JSON <query>;
  ```
- For Aurora PostgreSQL — check slow queries:
  ```sql
  -- Requires pg_stat_statements extension
  SELECT query, calls, mean_exec_time, total_exec_time,
         rows, shared_blks_hit, shared_blks_read
  FROM pg_stat_statements
  ORDER BY mean_exec_time DESC LIMIT 10;

  -- Explain a specific query
  EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) <query>;
  ```

SHOULD:
- Check for missing indexes:
  - Aurora MySQL:
    ```sql
    SELECT object_schema, object_name, index_name, count_star, sum_timer_wait
    FROM performance_schema.table_io_waits_summary_by_index_usage
    WHERE index_name IS NULL ORDER BY sum_timer_wait DESC LIMIT 10;
    ```
  - Aurora PostgreSQL:
    ```sql
    SELECT schemaname, relname, seq_scan, seq_tup_read, idx_scan
    FROM pg_stat_user_tables
    WHERE seq_scan > 100 AND idx_scan = 0 ORDER BY seq_tup_read DESC LIMIT 10;
    ```
- Check for stale statistics:
  - Aurora MySQL: `SHOW TABLE STATUS FROM <database>;`
  - Aurora PostgreSQL: `SELECT schemaname, relname, last_analyze, last_autoanalyze FROM pg_stat_user_tables;`

MAY:
- Enable auto_explain (PostgreSQL) for automatic plan logging
- Check for parameter sniffing issues (PostgreSQL: plan_cache_mode)
- Review Aurora parallel query eligibility (MySQL only)

## Phase 2 — Remediate

MUST:
- Add missing indexes for frequently scanned tables
- Update statistics: MySQL `ANALYZE TABLE`, PostgreSQL `ANALYZE`
- Rewrite inefficient queries (reduce rows examined, avoid SELECT *)

SHOULD:
- For Aurora MySQL: use query hints or optimizer hints if needed
- For Aurora PostgreSQL: reset pg_stat_statements periodically to track recent patterns
- Implement query timeout settings to prevent runaway queries

MAY:
- Enable Aurora parallel query (MySQL only) for analytical queries
- Use Aurora cloning to test index and query changes safely
- Consider read/write splitting for read-heavy query workloads

## Common Issues

- symptoms: "Query suddenly became slow"
  diagnosis: "Stale statistics, plan change, or data growth changed execution plan."
  resolution: "Update statistics. Check execution plan. Add indexes if needed."

- symptoms: "High rows_examined to rows_sent ratio"
  diagnosis: "Query scanning many rows but returning few. Missing or unused index."
  resolution: "Add appropriate index. Rewrite query to use existing indexes."

- symptoms: "Lock wait timeout exceeded (MySQL) or deadlock detected (PostgreSQL)"
  diagnosis: "Concurrent transactions competing for the same rows."
  resolution: "Optimize transaction scope. Reduce lock duration. Implement retry logic."

## Safety Ratings
- GREEN: Performance Insights queries, EXPLAIN/EXPLAIN ANALYZE, SELECT from performance_schema/pg_stat_statements, SHOW TABLE STATUS — read-only inspection
- YELLOW: ANALYZE TABLE, CREATE INDEX, enabling auto_explain/pg_stat_statements — recoverable but may impact performance during execution
- RED: force-failover, delete-db-instance, DROP INDEX on production — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires creating indexes on large production tables (may cause locks)"
- "Fix requires failover of Aurora cluster"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, full SQL query text from pg_stat_statements/performance_schema (may contain sensitive data in WHERE clauses)
- HIGH: execution plans (reveal table structure and data distribution)
- MEDIUM: table statistics, index usage metrics, Performance Insights summaries

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix query issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest dropping indexes in production without confirming they are truly unused"

## Phase 3 — Rollback
- "Restore from snapshot if index or parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If a new index causes write performance degradation, drop it and reassess"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "query_performance — <specific_cause>"
evidence:
  - type: performance_insights
    content: "<top SQL and wait events>"
  - type: execution_plan
    content: "<EXPLAIN output for problematic query>"
  - type: statistics
    content: "<table statistics and index usage>"
severity: HIGH
mitigation:
  immediate: "Optimize specific slow queries"
  long_term: "Implement query monitoring, index management, and statistics maintenance"
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
