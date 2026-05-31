---
title: "B3 — Lock Waits and Deadlocks"
description: "Diagnose InnoDB lock waits, deadlocks, and long-running transactions"
status: active
severity: HIGH
triggers:
  - "lock wait"
  - "deadlock"
  - "Lock wait timeout exceeded"
  - "long running transaction"
---

## Phase 1 — Triage

MUST:
- Check for active lock waits:
  ```sql
  SELECT r.trx_id AS waiting_trx, r.trx_mysql_thread_id AS waiting_thread,
         b.trx_id AS blocking_trx, b.trx_mysql_thread_id AS blocking_thread,
         r.trx_query AS waiting_query
  FROM information_schema.INNODB_LOCK_WAITS w
  JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
  JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id;
  ```
- Check for long-running transactions:
  ```sql
  SELECT trx_id, trx_state, trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
         trx_rows_locked, trx_rows_modified, trx_query
  FROM information_schema.INNODB_TRX ORDER BY trx_started;
  ```
- Check latest deadlock: `SHOW ENGINE INNODB STATUS\G` — look for LATEST DETECTED DEADLOCK section
- Check Performance Insights for lock wait events

SHOULD:
- Check `innodb_lock_wait_timeout` setting (default 50 seconds)
- Review `innodb_deadlock_detect` setting (ON by default)
- Check for uncommitted transactions from application connection leaks

## Phase 2 — Remediate

- Kill blocking sessions: `CALL mysql.rds_kill(<thread_id>);`
- Reduce transaction scope: commit frequently, avoid long-running transactions
- Add appropriate indexes to reduce lock scope (row-level vs table-level)
- For deadlocks: reorder operations to access tables/rows in consistent order
- Increase `innodb_lock_wait_timeout` if legitimate long transactions exist

## Safety Ratings
- GREEN: SELECT from information_schema.INNODB_LOCK_WAITS/INNODB_TRX, SHOW ENGINE INNODB STATUS, Performance Insights — read-only inspection
- YELLOW: CALL mysql.rds_kill(), modify-db-parameter-group (innodb_lock_wait_timeout) — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Lock contention causing widespread application blocking"
- "Fix requires killing active database sessions"
- "Deadlocks occurring frequently"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from INNODB_TRX (contain active queries)
- HIGH: SHOW ENGINE INNODB STATUS (contains deadlock details and SQL text)
- MEDIUM: lock wait statistics, transaction details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix lock issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest killing sessions without confirming with the application team"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "If killing a blocking session causes transaction rollback, wait for rollback to complete"
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
