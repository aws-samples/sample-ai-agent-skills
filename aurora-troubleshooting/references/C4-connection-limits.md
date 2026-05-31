---
title: "C4 — Connection Limits"
description: "Diagnose connection limit exhaustion on Aurora instances"
status: active
severity: HIGH
triggers:
  - "too many connections"
  - "max_connections"
  - "connection limit"
  - "connection pool exhausted"
  - "connection refused"
owner: devops-agent
objective: "Identify and resolve connection limit issues on Aurora instances"
context: "Aurora max_connections is determined by instance class memory. Each connection consumes memory. Connection exhaustion prevents new connections and can cause application outages. RDS Proxy is the recommended solution for connection management."
---

## Phase 1 — Triage

MUST:
- Check current connection count:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <start> --end-time <end> --period 60 --statistics Maximum
  ```
- Check max_connections setting:
  - Aurora MySQL: `SHOW VARIABLES LIKE 'max_connections';`
  - Aurora PostgreSQL: `SHOW max_connections;`
- Check current connections by user/host:
  - Aurora MySQL:
    ```sql
    SELECT user, host, COUNT(*) AS connections
    FROM information_schema.processlist GROUP BY user, host ORDER BY connections DESC;
    ```
  - Aurora PostgreSQL:
    ```sql
    SELECT usename, client_addr, state, COUNT(*)
    FROM pg_stat_activity GROUP BY usename, client_addr, state ORDER BY count DESC;
    ```

SHOULD:
- Check for idle connections:
  - Aurora MySQL: `SELECT COUNT(*) FROM information_schema.processlist WHERE command = 'Sleep' AND time > 300;`
  - Aurora PostgreSQL: `SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle' AND query_start < now() - interval '5 minutes';`
- Check connection trends over time:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <24h-ago> --end-time <now> --period 3600 --statistics Maximum
  ```
- Check FreeableMemory (connections consume memory):
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
  ```

MAY:
- Check if RDS Proxy is configured: `aws rds describe-db-proxies`
- Review application connection pool settings

## Phase 2 — Remediate

MUST:
- Implement connection pooling: RDS Proxy (recommended) or application-level pooling
- Kill idle connections if in emergency:
  - Aurora MySQL: `CALL mysql.rds_kill(<process_id>);`
  - Aurora PostgreSQL: `SELECT pg_terminate_backend(<pid>);`
- Increase max_connections if instance memory allows (parameter group change + reboot)

SHOULD:
- Configure application connection pool with appropriate min/max/idle timeout
- Set `wait_timeout` (MySQL) or `idle_in_transaction_session_timeout` (PostgreSQL) to close idle connections
- Scale up instance class for higher max_connections (based on memory)

MAY:
- Add reader instances to distribute read connections
- Implement connection queuing in the application layer

## Common Issues

- symptoms: "ERROR 1040: Too many connections (MySQL)"
  diagnosis: "Connection count reached max_connections limit."
  resolution: "Implement RDS Proxy. Kill idle connections. Scale up instance."

- symptoms: "FATAL: too many connections for role (PostgreSQL)"
  diagnosis: "Connection count exceeded max_connections or per-role connection limit."
  resolution: "Implement RDS Proxy. Check per-role limits with ALTER ROLE ... CONNECTION LIMIT."

- symptoms: "Connection pool exhausted in application"
  diagnosis: "Application pool max size reached. Connections not being returned to pool."
  resolution: "Fix connection leaks. Increase pool size. Reduce connection hold time."

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch DatabaseConnections/FreeableMemory metrics, SHOW VARIABLES, pg_stat_activity, information_schema.processlist — read-only inspection
- YELLOW: modify-db-parameter-group (max_connections), CALL mysql.rds_kill(), pg_terminate_backend() — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires killing active database sessions"
- "Fix requires failover of Aurora cluster"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, user/host connection details from processlist/pg_stat_activity
- HIGH: query text visible in processlist and pg_stat_activity (may contain sensitive data)
- MEDIUM: connection count metrics, max_connections settings, idle connection statistics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix connection issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest killing all connections without confirming with the application team"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes (max_connections, wait_timeout) and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "If killing idle connections causes application errors, investigate connection pool configuration"

## Output Format

```yaml
root_cause: "connection_limit — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<DatabaseConnections metric>"
  - type: connection_breakdown
    content: "<connections by user/host/state>"
  - type: max_connections
    content: "<max_connections setting>"
severity: HIGH
mitigation:
  immediate: "Kill idle connections or scale up instance"
  long_term: "Implement RDS Proxy and connection pool management"
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
