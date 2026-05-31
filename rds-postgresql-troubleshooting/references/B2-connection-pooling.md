---
title: "B2 — Connection Pooling Issues"
description: "Diagnose connection exhaustion and pooling configuration"
status: active
severity: HIGH
triggers:
  - "too many connections"
  - "max_connections"
  - "connection pooling"
  - "FATAL: too many connections"
---

## Phase 1 — Triage

MUST:
- Check connection count:
  ```sql
  SELECT count(*) AS total, state FROM pg_stat_activity GROUP BY state;
  SELECT usename, count(*) FROM pg_stat_activity GROUP BY usename ORDER BY count DESC;
  SHOW max_connections;
  ```
- Check DatabaseConnections metric: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=<id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check for idle connections:
  ```sql
  SELECT pid, usename, state, state_change, query
  FROM pg_stat_activity WHERE state = 'idle' AND state_change < NOW() - INTERVAL '10 minutes';
  ```

SHOULD:
- Check if RDS Proxy is in use: `aws rds describe-db-proxies`
- Check `idle_in_transaction_session_timeout` setting
- Check reserved connections: `superuser_reserved_connections` (default 3)

## Phase 2 — Remediate

- Kill idle connections: `SELECT pg_terminate_backend(<pid>);`
- Set `idle_in_transaction_session_timeout` to auto-kill stale transactions
- Deploy RDS Proxy for connection pooling: reduces backend connections significantly
- If using PgBouncer: verify pool_mode (transaction mode recommended for most apps)
- Scale instance class to increase max_connections (memory-based formula)
- Application fix: ensure connection pools release connections properly

## Safety Ratings
- GREEN: SELECT from pg_stat_activity, SHOW max_connections, CloudWatch DatabaseConnections — read-only inspection
- YELLOW: modify-db-parameter-group (max_connections), pg_terminate_backend() — recoverable but impacts sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Connection limit reached causing application outages"
- "Fix requires parameter group change (max_connections) that needs reboot"
- "Fix requires killing active database sessions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, pg_stat_activity details (contain SQL text and user info)
- MEDIUM: connection count metrics, max_connections settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix connection issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest killing all connections without confirming with the application team"

## Phase 3 — Rollback
- "Revert parameter group changes (max_connections) and reboot if needed"
- "If killing sessions causes application errors, investigate connection pool configuration"
- "Restore from snapshot if parameter change causes issues"

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
