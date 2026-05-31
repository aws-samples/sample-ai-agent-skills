---
title: "D3 — Session and Process Limits"
description: "Diagnose Oracle session and process limit issues on RDS"
status: active
severity: HIGH
triggers:
  - "ORA-00018"
  - "ORA-00020"
  - "maximum sessions"
  - "maximum processes"
  - "too many connections"
  - "process limit"
  - "session limit"
owner: devops-agent
objective: "Resolve session and process limit exhaustion on RDS Oracle"
context: "Oracle sessions = 1.5 * processes + 22 (approximately). The processes parameter is static (requires reboot). Connection pooling is critical for managing session counts. RDS has instance-class-based limits on top of Oracle limits."
---

## Phase 1 — Triage

MUST:
- Check current utilization:
  ```sql
  SELECT resource_name, current_utilization, max_utilization, limit_value
  FROM v$resource_limit
  WHERE resource_name IN ('sessions','processes','transactions');
  ```
- Check parameter settings:
  ```sql
  SELECT name, value FROM v$parameter WHERE name IN ('processes','sessions');
  ```
- Check active vs inactive sessions:
  ```sql
  SELECT status, COUNT(*) FROM v$session GROUP BY status;
  ```
- Check DatabaseConnections metric: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Identify top session consumers by username/program:
  ```sql
  SELECT username, program, machine, COUNT(*) AS sessions
  FROM v$session WHERE username IS NOT NULL
  GROUP BY username, program, machine ORDER BY sessions DESC;
  ```
- Find long-idle sessions:
  ```sql
  SELECT sid, serial#, username, program, status,
         last_call_et/60 AS idle_minutes
  FROM v$session WHERE status = 'INACTIVE' AND username IS NOT NULL
  ORDER BY last_call_et DESC FETCH FIRST 20 ROWS ONLY;
  ```
- Check for connection leaks (sessions growing over time)

MAY:
- Check Oracle profiles for session limits:
  ```sql
  SELECT profile, resource_name, limit FROM dba_profiles
  WHERE resource_name IN ('SESSIONS_PER_USER','IDLE_TIME','CONNECT_TIME');
  ```

## Phase 2 — Remediate

MUST:
- Kill idle sessions to free up immediately:
  ```sql
  EXEC rdsadmin.rdsadmin_util.kill('<sid>', '<serial#>');
  ```
- Increase processes parameter (requires reboot):
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=processes,ParameterValue=<new_value>,ApplyMethod=pending-reboot"
  aws rds reboot-db-instance --db-instance-identifier <instance-id>
  ```

SHOULD:
- Implement connection pooling in the application
- Set Oracle profile limits to auto-disconnect idle sessions:
  ```sql
  ALTER PROFILE DEFAULT LIMIT IDLE_TIME 30;
  ALTER PROFILE DEFAULT LIMIT SESSIONS_PER_USER 50;
  ```
- Set up CloudWatch alarm on DatabaseConnections

MAY:
- Use Oracle Resource Manager to limit sessions per consumer group
- Implement application-level connection health checks

## Common Issues

- symptoms: "ORA-00020: maximum number of processes (150) exceeded"
  diagnosis: "processes parameter too low. Default is often 150."
  resolution: "Increase processes in parameter group. Reboot required. Also implement connection pooling."

- symptoms: "ORA-00018: maximum number of sessions exceeded"
  diagnosis: "sessions limit reached. sessions ≈ 1.5 * processes + 22."
  resolution: "Increase processes (sessions adjusts automatically). Kill idle sessions."

- symptoms: "Connections growing steadily, never decreasing"
  diagnosis: "Connection leak in application. Sessions not being closed."
  resolution: "Fix application connection management. Set IDLE_TIME profile limit."

## Safety Ratings
- GREEN: SELECT from v$resource_limit/v$session/v$parameter, CloudWatch DatabaseConnections metrics — read-only inspection
- YELLOW: EXEC rdsadmin.rdsadmin_util.kill, ALTER PROFILE, modify-db-parameter-group (processes) — recoverable but impacts sessions or requires reboot
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Session/process limit reached causing application outages"
- "Fix requires increasing processes parameter (needs reboot)"
- "Fix requires killing active database sessions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, v$session details (contain username, program, machine, SQL text)
- HIGH: Oracle profile settings (reveal security configuration)
- MEDIUM: resource limit utilization, connection count metrics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix session limit issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest killing all sessions without confirming with the application team"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes (processes) and reboot if needed"
- "If killing sessions causes application errors, investigate connection pool configuration"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "session_process_limit — <specific_cause>"
evidence:
  - type: resource_limit
    content: "<v$resource_limit output>"
  - type: session_breakdown
    content: "<sessions by username/program>"
severity: HIGH
mitigation:
  immediate: "Kill idle sessions"
  long_term: "Increase processes, implement connection pooling, set profile limits"
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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
