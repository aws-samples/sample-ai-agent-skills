---
title: "B1 — Vacuum and Table Bloat"
description: "Diagnose autovacuum issues, table bloat, and transaction ID wraparound"
status: active
severity: CRITICAL
triggers:
  - "vacuum"
  - "autovacuum"
  - "bloat"
  - "dead tuples"
  - "wraparound"
  - "n_dead_tup"
---

## Phase 1 — Triage

MUST:
- Check dead tuple counts:
  ```sql
  SELECT schemaname, relname, n_live_tup, n_dead_tup,
         ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
         last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
  FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;
  ```
- Check transaction ID age (wraparound risk):
  ```sql
  SELECT datname, age(datfrozenxid) AS xid_age,
         ROUND(age(datfrozenxid)::numeric / 2000000000 * 100, 2) AS pct_to_wraparound
  FROM pg_database ORDER BY age(datfrozenxid) DESC;
  ```
- Check autovacuum workers:
  ```sql
  SELECT pid, datname, relid::regclass, phase, heap_blks_total, heap_blks_scanned
  FROM pg_stat_progress_vacuum;
  ```
- Check autovacuum settings: `aws rds describe-db-parameters --db-parameter-group-name <group> --query 'Parameters[?starts_with(ParameterName,\`autovacuum\`)]'`

SHOULD:
- Check table bloat estimate:
  ```sql
  SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size
  FROM pg_tables WHERE schemaname = 'public' ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC LIMIT 20;
  ```
- Check if autovacuum is being cancelled by conflicting locks

## Phase 2 — Remediate

- High dead tuples: run manual VACUUM on affected tables `VACUUM (VERBOSE) <table>;`
- Bloated tables: `VACUUM FULL <table>;` (requires exclusive lock — schedule during maintenance)
- Tune autovacuum per-table for high-churn tables:
  ```sql
  ALTER TABLE <table> SET (autovacuum_vacuum_scale_factor = 0.01, autovacuum_vacuum_threshold = 1000);
  ```
- Wraparound risk: increase `autovacuum_max_workers`, reduce `autovacuum_vacuum_cost_delay`
- Index bloat: `REINDEX CONCURRENTLY INDEX <index>;`

## Safety Ratings
- GREEN: SELECT from pg_stat_user_tables/pg_stat_activity, SHOW autovacuum settings — read-only inspection
- YELLOW: VACUUM (standard), ANALYZE, modify-db-parameter-group (autovacuum settings) — recoverable but may impact performance
- RED: VACUUM FULL (causes exclusive table lock), delete-db-instance — high-impact or destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "VACUUM FULL required on large production tables (causes exclusive lock)"
- "Transaction ID wraparound approaching (autovacuum_freeze_max_age)"
- "Fix requires parameter group change that needs reboot"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings
- MEDIUM: table bloat statistics, autovacuum configuration, dead tuple counts

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix vacuum issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest running VACUUM FULL on large production tables without confirming maintenance window"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if vacuum parameter change causes issues"
- "If VACUUM FULL causes performance degradation, wait for completion — do NOT kill the process"
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
