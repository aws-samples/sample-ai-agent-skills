---
title: "B1 — High CPU / Wait Events"
description: "Diagnose high CPU utilization and wait events on Aurora instances"
status: active
severity: HIGH
triggers:
  - "high CPU"
  - "CPUUtilization"
  - "wait events"
  - "slow queries"
  - "db.load"
  - "performance degradation"
owner: devops-agent
objective: "Identify and resolve high CPU utilization and wait event issues on Aurora"
context: "Aurora CPU issues can stem from inefficient queries, missing indexes, lock contention, or undersized instances. Wait events differ between Aurora MySQL (performance_schema) and Aurora PostgreSQL (pg_stat_activity). Performance Insights provides unified DB load analysis."
---

## Phase 1 — Triage

MUST:
- Check CPU metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Average Maximum
  ```
- Check DB load via Performance Insights:
  ```
  aws pi get-resource-metrics --service-type RDS --identifier db-<resource-id> \
    --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.wait_event"}}]' \
    --start-time <start> --end-time <end> --period-in-seconds 300
  ```
- For Aurora MySQL — check active threads and waits:
  ```sql
  SELECT thread_id, processlist_user, processlist_command, processlist_state,
         processlist_time, processlist_info
  FROM performance_schema.threads
  WHERE processlist_command != 'Sleep' AND type = 'FOREGROUND'
  ORDER BY processlist_time DESC;

  SELECT event_name, count_star, sum_timer_wait/1000000000 AS total_wait_ms
  FROM performance_schema.events_waits_summary_global_by_event_name
  WHERE count_star > 0 ORDER BY sum_timer_wait DESC LIMIT 20;
  ```
- For Aurora PostgreSQL — check active queries and waits:
  ```sql
  SELECT pid, usename, state, wait_event_type, wait_event,
         now() - query_start AS duration, query
  FROM pg_stat_activity
  WHERE state = 'active' AND pid != pg_backend_pid()
  ORDER BY duration DESC;
  ```

SHOULD:
- Check top SQL by CPU:
  - Aurora MySQL:
    ```sql
    SELECT digest_text, count_star, sum_timer_wait/1000000000 AS total_ms,
           sum_rows_examined, sum_rows_sent
    FROM performance_schema.events_statements_summary_by_digest
    ORDER BY sum_timer_wait DESC LIMIT 10;
    ```
  - Aurora PostgreSQL:
    ```sql
    SELECT query, calls, total_exec_time, mean_exec_time, rows
    FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
    ```
- Check for lock contention:
  - Aurora MySQL: `SELECT * FROM performance_schema.data_locks WHERE lock_status = 'WAITING';`
  - Aurora PostgreSQL: `SELECT * FROM pg_locks WHERE NOT granted;`

MAY:
- Check Enhanced Monitoring for OS-level CPU breakdown
- Review query execution plans for top CPU consumers

## Phase 2 — Remediate

MUST:
- Identify and optimize top CPU-consuming queries (add indexes, rewrite queries)
- For lock contention: identify blocking sessions and resolve deadlocks
- Scale up instance class if CPU is consistently saturated after query optimization

SHOULD:
- Enable Performance Insights if not already enabled (free tier: 7 days retention)
- For Aurora MySQL: enable slow query log (`slow_query_log = 1`, `long_query_time = 1`)
- For Aurora PostgreSQL: enable `pg_stat_statements` extension and set `log_min_duration_statement`

MAY:
- Consider read/write splitting to offload reads to reader instances
- Use Aurora parallel query (MySQL only) for analytical workloads

## Common Issues

- symptoms: "CPU at 100% on writer instance"
  diagnosis: "Inefficient queries, missing indexes, or lock contention."
  resolution: "Identify top SQL via Performance Insights. Optimize queries. Add indexes."

- symptoms: "High db.load with wait event 'io/aurora_redo_log_flush'"
  diagnosis: "Write-heavy workload waiting on redo log flushes."
  resolution: "Batch commits. Reduce write frequency. Scale up instance class."

- symptoms: "CPU spikes on reader instances"
  diagnosis: "Heavy read queries or replication apply overhead."
  resolution: "Optimize read queries. Add more readers. Use custom endpoints for workload isolation."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-instances, CloudWatch CPUUtilization/db.load metrics, Performance Insights queries, SELECT from performance_schema/pg_stat_activity — read-only inspection
- YELLOW: modify-db-instance (scale up), enabling slow_query_log/pg_stat_statements — recoverable changes
- RED: force-failover, delete-db-instance — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "CPU sustained above 90% affecting application response times"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, query text from performance_schema/pg_stat_activity (contain SQL with potential sensitive data)
- HIGH: query results from performance_schema.threads, pg_stat_activity (contain active SQL text)
- MEDIUM: CloudWatch CPU metrics, Performance Insights DB load, wait event summaries

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix CPU issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest killing long-running queries in production without confirming with the application team"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "If instance scale-up causes issues, scale back down after confirming application compatibility"

## Output Format

```yaml
root_cause: "high_cpu — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<CPUUtilization metrics>"
  - type: performance_insights
    content: "<DB load and top wait events>"
  - type: top_sql
    content: "<top CPU-consuming queries>"
severity: HIGH
mitigation:
  immediate: "Optimize top CPU queries or scale up instance"
  long_term: "Implement query performance monitoring and index management"
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
