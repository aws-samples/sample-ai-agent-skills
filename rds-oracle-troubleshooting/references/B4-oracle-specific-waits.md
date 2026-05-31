---
title: "B4 — Oracle-Specific Wait Events"
description: "Diagnose Oracle-specific wait events beyond CPU and I/O"
status: active
severity: MEDIUM
triggers:
  - "enqueue"
  - "latch"
  - "library cache"
  - "row lock"
  - "TX enqueue"
  - "cursor pin"
  - "buffer busy"
  - "gc"
owner: devops-agent
objective: "Identify and resolve Oracle-specific wait events causing performance degradation"
context: "Oracle wait events beyond CPU and I/O include enqueue waits (locks), latch contention, library cache issues, buffer busy waits, and concurrency waits. These require Oracle-specific diagnostics via v$session, v$lock, and v$latch views."
---

## Phase 1 — Triage

MUST:
- Check active wait events:
  ```sql
  SELECT wait_class, event, COUNT(*) AS sessions,
         SUM(seconds_in_wait) AS total_wait_sec
  FROM v$session WHERE status = 'ACTIVE' AND wait_class != 'Idle'
  GROUP BY wait_class, event ORDER BY sessions DESC;
  ```
- Check system-level wait statistics:
  ```sql
  SELECT wait_class, event, total_waits, time_waited/100 AS time_sec,
         average_wait/100 AS avg_wait_sec
  FROM v$system_event WHERE wait_class NOT IN ('Idle','Other')
  ORDER BY time_waited DESC FETCH FIRST 20 ROWS ONLY;
  ```
- Check for lock contention:
  ```sql
  SELECT l.sid AS blocker_sid, s.username AS blocker_user,
         l2.sid AS waiter_sid, s2.username AS waiter_user,
         l.type, o.object_name
  FROM v$lock l
  JOIN v$lock l2 ON l.id1 = l2.id1 AND l.id2 = l2.id2 AND l.block = 1 AND l2.request > 0
  JOIN v$session s ON l.sid = s.sid
  JOIN v$session s2 ON l2.sid = s2.sid
  LEFT JOIN v$locked_object lo ON l.sid = lo.session_id
  LEFT JOIN dba_objects o ON lo.object_id = o.object_id;
  ```

SHOULD:
- Check latch contention:
  ```sql
  SELECT name, gets, misses, sleeps, immediate_gets, immediate_misses
  FROM v$latch ORDER BY sleeps DESC FETCH FIRST 10 ROWS ONLY;
  ```
- Check library cache contention:
  ```sql
  SELECT namespace, gets, gethits, pins, pinhits,
         ROUND(gethits/NULLIF(gets,0)*100,2) AS get_hit_pct
  FROM v$librarycache;
  ```
- Check buffer busy waits:
  ```sql
  SELECT file#, block#, class#, COUNT(*) AS waits
  FROM v$waitstat GROUP BY file#, block#, class# ORDER BY waits DESC;
  ```

MAY:
- Use ASH (Active Session History) for historical analysis (EE):
  ```sql
  SELECT event, COUNT(*) AS samples FROM v$active_session_history
  WHERE sample_time > SYSDATE - INTERVAL '1' HOUR
  GROUP BY event ORDER BY samples DESC;
  ```
- Check Performance Insights for wait event trends

## Phase 2 — Remediate

MUST:
- For TX enqueue (row lock): identify and resolve blocking sessions
  ```sql
  -- Kill blocking session (via rdsadmin)
  EXEC rdsadmin.rdsadmin_util.kill('<sid>', '<serial#>');
  ```
- For library cache waits: use bind variables, increase shared pool
- For buffer busy waits: reduce hot block contention, increase buffer cache

SHOULD:
- For latch contention: reduce concurrent access to shared structures
- For cursor pin S waits: check for excessive version count on SQL
  ```sql
  SELECT sql_id, COUNT(*) AS versions FROM v$sql GROUP BY sql_id HAVING COUNT(*) > 10 ORDER BY versions DESC;
  ```
- Implement application-level retry logic for lock timeouts

MAY:
- Use Oracle Resource Manager to limit resource consumption per session
- Implement row-level locking strategies in application code

## Common Issues

- symptoms: "Sessions waiting on 'enq: TX - row lock contention'"
  diagnosis: "Multiple sessions trying to update the same rows."
  resolution: "Identify blocker with v$lock. Kill if appropriate. Fix application locking order."

- symptoms: "High 'library cache: mutex X' waits"
  diagnosis: "Hard parsing due to literal SQL (no bind variables)."
  resolution: "Use bind variables. Increase session_cached_cursors. Consider cursor_sharing=FORCE."

- symptoms: "'buffer busy waits' on specific blocks"
  diagnosis: "Hot blocks — many sessions reading/writing the same data blocks."
  resolution: "Reduce contention via partitioning (EE), reverse key indexes, or application changes."

## Safety Ratings
- GREEN: SELECT from v$session/v$system_event/v$lock/v$latch/v$librarycache, Performance Insights — read-only inspection
- YELLOW: EXEC rdsadmin.rdsadmin_util.kill (kill blocking session), ALTER SYSTEM FLUSH SHARED_POOL — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Lock contention causing widespread application blocking"
- "Fix requires killing production sessions"
- "Fix requires parameter group change that needs reboot"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from v$session/v$sql (contain active queries with potential sensitive data)
- HIGH: v$lock and v$session details (reveal blocking chains and user activity)
- MEDIUM: latch statistics, library cache hit ratios, buffer busy wait details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix wait event issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest cursor_sharing=FORCE without exhausting other options first"
- "NEVER suggest killing sessions without confirming with the application team"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If killing a blocking session causes transaction rollback, wait for rollback to complete"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "oracle_wait — <specific_wait_event>: <cause>"
evidence:
  - type: wait_events
    content: "<v$session wait event summary>"
  - type: locks
    content: "<v$lock blocker/waiter chain>"
severity: MEDIUM
mitigation:
  immediate: "Kill blocking sessions or flush shared pool"
  long_term: "Fix application SQL patterns and locking strategy"
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
