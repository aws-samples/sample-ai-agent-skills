---
title: "B1 — RDS MySQL Slow Queries"
description: "Diagnose and resolve slow query performance on RDS MySQL"
status: active
severity: HIGH
triggers:
  - "slow queries"
  - "query timeout"
  - "high CPU"
  - "long running queries"
---

## Phase 1 — Triage

MUST:
- Check CPU and DB load: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check Performance Insights for top SQL: `aws pi get-resource-metrics --service-type RDS --identifier db-<resource-id> --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.sql","Limit":10}}]' --start-time <start> --end-time <end> --period-in-seconds 300`
- Enable slow query log if not enabled: check parameter `slow_query_log=1`, `long_query_time=2`
- Review slow log: `aws rds download-db-log-file-portion --db-instance-identifier <id> --log-file-name slowquery/mysql-slowquery.log`

SHOULD:
- Check for missing indexes:
  ```sql
  EXPLAIN SELECT ... ; -- for the slow query
  SHOW INDEX FROM <table>;
  SELECT * FROM sys.statements_with_full_table_scans ORDER BY no_index_used_count DESC LIMIT 10;
  ```
- Check for table locks:
  ```sql
  SHOW OPEN TABLES WHERE In_use > 0;
  SELECT * FROM performance_schema.data_lock_waits;
  ```
- Review query execution plans for full table scans

## Phase 2 — Remediate

- Add missing indexes based on EXPLAIN output
- Optimize queries: avoid SELECT *, use LIMIT, rewrite subqueries as JOINs
- Increase `innodb_buffer_pool_size` if hit ratio < 95% (via parameter group)
- Scale instance class if CPU consistently > 80%
- Use read replicas to offload read-heavy queries

## Safety Ratings
- GREEN: CloudWatch CPUUtilization metrics, Performance Insights, EXPLAIN, SHOW INDEX, slow query log — read-only inspection
- YELLOW: CREATE INDEX, modify-db-parameter-group (slow_query_log, long_query_time) — recoverable but may impact performance
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires creating indexes on large production tables"
- "Fix requires parameter group change that needs reboot"
- "CPU sustained above 90% affecting application response times"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, slow query log (contains full SQL text with potential sensitive data)
- HIGH: EXPLAIN output (reveals table structure and data distribution)
- MEDIUM: CloudWatch CPU metrics, Performance Insights data

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix query issues — use snapshots"
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
