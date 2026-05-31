---
title: "B2 — Memory Pressure (SGA/PGA)"
description: "Diagnose Oracle SGA and PGA memory pressure on RDS Oracle"
status: active
severity: HIGH
triggers:
  - "ORA-04031"
  - "shared pool"
  - "SGA"
  - "PGA"
  - "FreeableMemory"
  - "memory pressure"
  - "swap usage"
  - "out of memory"
owner: devops-agent
objective: "Identify and resolve Oracle memory allocation issues (SGA/PGA) on RDS"
context: "RDS manages Oracle memory based on instance class. SGA includes shared pool, buffer cache, large pool, and java pool. PGA is per-session work area memory. Over-allocation causes ORA-04031 or instance instability. memory_target (AMM) is not supported on RDS — use ASMM (sga_target + pga_aggregate_target)."
---

## Phase 1 — Triage

MUST:
- Check FreeableMemory: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check SwapUsage: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name SwapUsage ...`
- Check SGA allocation:
  ```sql
  SELECT component, current_size/1024/1024 AS current_mb,
         min_size/1024/1024 AS min_mb, max_size/1024/1024 AS max_mb
  FROM v$sga_dynamic_components ORDER BY current_size DESC;
  ```
- Check PGA usage:
  ```sql
  SELECT name, value/1024/1024 AS mb FROM v$pgastat
  WHERE name IN ('total PGA allocated','maximum PGA allocated',
                 'total PGA inuse','global memory bound');
  ```
- Check parameter settings:
  ```sql
  SELECT name, value FROM v$parameter
  WHERE name IN ('sga_target','sga_max_size','pga_aggregate_target',
                 'shared_pool_size','db_cache_size','memory_target');
  ```

SHOULD:
- Check for ORA-04031 in alert log: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name trace/alert_<SID>.log --number-of-lines 500`
- Check shared pool fragmentation:
  ```sql
  SELECT pool, name, bytes/1024/1024 AS mb FROM v$sgastat
  WHERE pool = 'shared pool' ORDER BY bytes DESC FETCH FIRST 10 ROWS ONLY;
  ```
- Check PGA per session:
  ```sql
  SELECT s.sid, s.serial#, s.username, p.pga_alloc_mem/1024/1024 AS pga_mb
  FROM v$session s JOIN v$process p ON s.paddr = p.addr
  WHERE s.username IS NOT NULL ORDER BY p.pga_alloc_mem DESC FETCH FIRST 10 ROWS ONLY;
  ```

MAY:
- Check for memory advisors: `SELECT * FROM v$sga_target_advice;`
- Check PGA advice: `SELECT * FROM v$pga_target_advice;`

## Phase 2 — Remediate

MUST:
- For ORA-04031: flush shared pool as emergency measure (causes brief performance dip)
  ```sql
  ALTER SYSTEM FLUSH SHARED_POOL;
  ```
- Adjust SGA/PGA via parameter group (total must not exceed ~75% of instance memory):
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=sga_target,ParameterValue=<bytes>,ApplyMethod=pending-reboot"
  ```
- For persistent memory pressure: scale to a larger instance class

SHOULD:
- Use bind variables to reduce shared pool fragmentation
- Reduce `open_cursors` if sessions hold too many open cursors
- Set `pga_aggregate_target` appropriately for sort/hash operations

MAY:
- Pin frequently used packages in shared pool: `EXEC DBMS_SHARED_POOL.KEEP('<package>');`
- Implement connection pooling to reduce total PGA consumption

## Common Issues

- symptoms: "ORA-04031: unable to allocate shared memory"
  diagnosis: "Shared pool fragmented or undersized. Often caused by non-bind-variable SQL."
  resolution: "Flush shared pool (emergency). Use bind variables. Increase sga_target or shared_pool_size."

- symptoms: "FreeableMemory near zero, SwapUsage increasing"
  diagnosis: "SGA + PGA exceeds instance memory. Instance is swapping."
  resolution: "Reduce sga_target or pga_aggregate_target. Scale to larger instance class."

- symptoms: "Temp tablespace sorts spilling to disk"
  diagnosis: "PGA work area too small for sort operations."
  resolution: "Increase pga_aggregate_target. Check for queries with large sorts."

## Safety Ratings
- GREEN: CloudWatch FreeableMemory/SwapUsage metrics, SELECT from v$sga_dynamic_components/v$pgastat/v$parameter — read-only inspection
- YELLOW: modify-db-parameter-group (sga_target, pga_aggregate_target), ALTER SYSTEM FLUSH SHARED_POOL — recoverable but may require reboot or cause brief performance dip
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "ORA-04031 errors occurring — shared pool exhaustion"
- "Fix requires parameter group change that needs reboot"
- "FreeableMemory critically low with swap usage increasing"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SGA/PGA configuration values
- HIGH: query results from v$session/v$process (contain session details and PGA allocation)
- MEDIUM: CloudWatch memory metrics, SGA component sizes, alert log contents

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix memory issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting memory_target on RDS Oracle — it is not supported"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest reducing SGA below minimum required for Oracle operation"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes (sga_target, pga_aggregate_target) and reboot if needed"
- "If FLUSH SHARED_POOL causes performance degradation, it will recover as cache warms up"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "memory_pressure — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<FreeableMemory, SwapUsage metrics>"
  - type: sga_config
    content: "<v$sga_dynamic_components output>"
  - type: pga_stats
    content: "<v$pgastat output>"
severity: HIGH
mitigation:
  immediate: "Flush shared pool or kill memory-heavy sessions"
  long_term: "Right-size SGA/PGA or scale instance class"
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
