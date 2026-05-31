---
title: "A3 — Lock Contention"
description: "Diagnose and resolve lock contention issues in Redshift"
status: active
severity: HIGH
triggers:
  - "lock contention"
  - "table locked"
  - "waiting for lock"
  - "deadlock"
  - "blocked query"
  - "AccessExclusiveLock"
owner: devops-agent
objective: "Identify lock holders and blocked queries, resolve contention"
context: "Redshift uses table-level locks for DDL and write operations. Long-running transactions, uncommitted writes, and concurrent DDL/DML can cause lock contention. Unlike row-level locking databases, Redshift locks entire tables for writes."
---

## Phase 1 — Triage

MUST:
- Check current locks: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT t.xid, t.pid, t.txn_owner, t.lock_mode, t.relation, t.granted, l.query, l.starttime FROM svv_transactions t JOIN stv_recents l ON t.pid = l.pid WHERE t.lockable_object_type = 'relation' ORDER BY t.granted, t.starttime"`
- Check blocked queries (granted = false): `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM svv_transactions WHERE granted = 'f'"`
- Check long-running transactions: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT xid, pid, txn_owner, txn_start, lock_mode FROM svv_transactions WHERE txn_start < DATEADD(minute, -10, GETDATE()) ORDER BY txn_start"`
- Identify the blocking query: correlate the lock holder's PID with stv_recents

SHOULD:
- Check if VACUUM or ANALYZE is holding locks
- Check for uncommitted transactions (BEGIN without COMMIT)
- Verify if DDL operations (ALTER TABLE, DROP) are causing AccessExclusiveLock

MAY:
- Check stl_tr_conflict for serializable isolation conflicts
- Review application connection pooling behavior
- Check for abandoned sessions holding locks

## Phase 2 — Remediate

MUST:
- If long-running transaction holding lock: `CANCEL <pid>` or `SELECT pg_terminate_backend(<pid>)`
- If uncommitted transaction: commit or rollback the transaction
- If DDL blocking DML: schedule DDL during maintenance windows

SHOULD:
- Implement application-level query timeouts
- Use `SET lock_timeout` to prevent indefinite lock waits
- Avoid long-running explicit transactions (BEGIN/COMMIT blocks)

MAY:
- Implement retry logic in applications for lock timeout errors
- Schedule VACUUM and DDL operations during off-peak hours
- Monitor svv_transactions for lock patterns

## Common Issues

- symptoms: "SELECT queries blocked waiting for AccessExclusiveLock"
  diagnosis: "A DDL operation (ALTER TABLE, VACUUM FULL) holds an exclusive lock."
  resolution: "Wait for DDL to complete or cancel it. Schedule DDL during maintenance windows."

- symptoms: "Multiple queries waiting, one transaction started hours ago"
  diagnosis: "Abandoned transaction holding locks — likely an uncommitted BEGIN block."
  resolution: "Terminate the session: SELECT pg_terminate_backend(<pid>). Fix application to use autocommit or proper transaction management."

- symptoms: "COPY command blocked by concurrent SELECT"
  diagnosis: "COPY acquires a write lock; concurrent long-running SELECTs may delay lock acquisition."
  resolution: "COPY should not be blocked by SELECT in normal operation. Check for explicit LOCK TABLE commands or serializable isolation."

## Output Format

```yaml
root_cause: "Lock contention — <specific_cause>"
evidence:
  - type: system_table
    content: "<svv_transactions lock details>"
severity: HIGH
mitigation:
  immediate: "Terminate blocking session or cancel blocking query"
  long_term: "Implement query timeouts, schedule DDL during off-peak, fix transaction management"
```


## Safety Ratings
```
safety_ratings:
  - "Query svv_transactions: GREEN — read-only system table query"
  - "Check stv_recents for running queries: GREEN — read-only inspection"
  - "CANCEL a blocking query: YELLOW — terminates query, transaction may need rollback"
  - "pg_terminate_backend: RED — forcefully terminates session, uncommitted work is lost"
  - "Set lock_timeout: GREEN — client-side configuration, no cluster impact"
```

## Escalation Conditions
- Cluster serves production analytics
- Lock contention blocking critical business queries or ETL pipelines
- Abandoned transactions holding locks for extended periods
- DDL operations blocking all DML on production tables
- Deadlock detected requiring manual intervention

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Transaction details: query text and user session info"
    - "Lock holder PIDs: session identifiers"
    - "Query text in stv_recents: may contain sensitive SQL"
  handling: "Do not expose session PIDs or query text externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER terminate sessions without confirming the transaction can be safely rolled back
- NEVER use pg_terminate_backend on system processes
- NEVER schedule DDL operations during peak query hours without a maintenance window

## Phase 3 — Rollback
- If a query was cancelled via CANCEL: the query's transaction is rolled back automatically
- If a session was terminated via pg_terminate_backend: the session's transaction is rolled back, user must reconnect
- If lock_timeout was set: remove the setting to restore default behavior
- If DDL was cancelled mid-operation: verify table state — some DDL operations are atomic, others may leave partial changes

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
