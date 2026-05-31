---
title: "B1 — High CPU and Wait Events"
description: "Diagnose high CPU utilization and Oracle wait events on RDS Oracle"
status: active
severity: HIGH
triggers:
  - "high CPU"
  - "CPUUtilization"
  - "slow queries"
  - "wait events"
  - "db load"
  - "performance degradation"
owner: devops-agent
objective: "Identify the root cause of high CPU and correlate with Oracle wait events"
context: "High CPU on RDS Oracle is typically caused by inefficient SQL, missing indexes, excessive parsing, or PL/SQL loops. Performance Insights provides DB load breakdown by wait events. Oracle wait events are the primary diagnostic tool for performance issues."
---

## Phase 1 — Triage

MUST:
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check DB load via Performance Insights:
  ```
  aws pi get-resource-metrics --service-type RDS \
    --identifier db-<resource-id> \
    --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.wait_event"}}]' \
    --start-time <start> --end-time <end> --period-in-seconds 300
  ```
- Identify top wait events via SQL:
  ```sql
  SELECT event, wait_class, COUNT(*) AS sessions, SUM(seconds_in_wait) AS total_wait
  FROM v$session WHERE status = 'ACTIVE' AND wait_class != 'Idle'
  GROUP BY event, wait_class ORDER BY sessions DESC;
  ```
- Identify top SQL by CPU:
  ```sql
  SELECT sql_id, cpu_time/1000000 AS cpu_sec, elapsed_time/1000000 AS elapsed_sec,
         executions, buffer_gets, disk_reads
  FROM v$sql ORDER BY cpu_time DESC FETCH FIRST 10 ROWS ONLY;
  ```

SHOULD:
- Check for full table scans: `SELECT sql_id, plan_hash_value FROM v$sql_plan WHERE operation = 'TABLE ACCESS' AND options = 'FULL' AND object_owner NOT IN ('SYS','SYSTEM');`
- Check for excessive parsing: `SELECT name, value FROM v$sysstat WHERE name IN ('parse count (total)','parse count (hard)','execute count');`
- Review execution plans for top SQL: `SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('<sql_id>'));`
- Check for lock contention: `SELECT * FROM v$lock WHERE block = 1;`

MAY:
- Enable Statspack snapshot for historical analysis (SE2): `EXEC statspack.snap;`
- Check AWR report (EE with Diagnostics Pack): `SELECT * FROM TABLE(DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_TEXT(l_dbid, l_inst, l_begin_snap, l_end_snap));`

## Phase 2 — Remediate

MUST:
- For inefficient SQL: add missing indexes, rewrite queries, update statistics
  ```sql
  EXEC DBMS_STATS.GATHER_TABLE_STATS('<owner>', '<table>');
  ```
- For excessive parsing: use bind variables, increase `open_cursors` and `session_cached_cursors`
- For CPU-bound PL/SQL: optimize loops, reduce context switches

SHOULD:
- Create SQL profiles or baselines for critical queries
- Scale instance class if CPU is legitimately saturated by workload
- Implement connection pooling to reduce session overhead

MAY:
- Use SQL Tuning Advisor (EE with Tuning Pack)
- Implement Resource Manager to limit CPU per session

## Common Issues

- symptoms: "CPU at 100%, top wait event is 'CPU + CPU Wait'"
  diagnosis: "SQL consuming excessive CPU. Check v$sql for top CPU consumers."
  resolution: "Optimize top SQL statements. Add indexes. Gather fresh statistics."

- symptoms: "High CPU with 'library cache: mutex X' waits"
  diagnosis: "Excessive hard parsing. Application not using bind variables."
  resolution: "Use bind variables. Increase session_cached_cursors. Consider cursor_sharing=FORCE as last resort."

## Safety Ratings
- GREEN: CloudWatch CPUUtilization metrics, Performance Insights, SELECT from v$session/v$sql/v$sql_plan — read-only inspection
- YELLOW: EXEC DBMS_STATS.GATHER_TABLE_STATS, CREATE INDEX, modify-db-parameter-group — recoverable but may impact performance
- RED: ALTER SYSTEM FLUSH SHARED_POOL (brief performance dip), delete-db-instance, force-failover — high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "CPU sustained above 90% affecting application response times"
- "Fix requires parameter group change that needs reboot"
- "Fix requires creating indexes on large production tables"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from v$sql/v$session (may contain sensitive data in WHERE clauses)
- HIGH: query results from v$session (contain active SQL text and session details)
- MEDIUM: CloudWatch CPU metrics, Performance Insights DB load, execution plans

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix CPU issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest cursor_sharing=FORCE without exhausting other options first"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If a new index causes write performance degradation, drop it and reassess"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "high_cpu — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<CPUUtilization metrics>"
  - type: wait_events
    content: "<top wait events from v$session>"
  - type: top_sql
    content: "<top SQL by CPU from v$sql>"
severity: HIGH
mitigation:
  immediate: "Optimize or kill the offending SQL"
  long_term: "Implement SQL monitoring and index strategy"
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
