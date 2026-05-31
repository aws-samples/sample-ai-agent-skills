---
title: "B2 — InnoDB Buffer Pool Issues"
description: "Diagnose InnoDB buffer pool pressure and tuning on RDS MySQL"
status: active
severity: HIGH
triggers:
  - "buffer pool"
  - "low hit ratio"
  - "FreeableMemory low"
  - "innodb_buffer_pool"
---

## Phase 1 — Triage

MUST:
- Check FreeableMemory: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory --dimensions Name=DBInstanceIdentifier,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check SwapUsage: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name SwapUsage ...`
- Check buffer pool hit ratio:
  ```sql
  SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
  -- Hit ratio = (1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests) * 100
  -- Target: > 95%
  ```
- Check buffer pool size vs data size:
  ```sql
  SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
  SELECT SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 AS total_mb FROM information_schema.TABLES WHERE ENGINE = 'InnoDB';
  ```

SHOULD:
- Check buffer pool pages:
  ```sql
  SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages%';
  -- pages_free near 0 = pressure
  ```
- Review `innodb_buffer_pool_instances` (should be 8 for pools > 1GB)

## Phase 2 — Remediate

- If hit ratio < 95%: increase instance class (buffer pool scales with memory)
- Default is `{DBInstanceClassMemory*3/4}` — do not exceed this
- If SwapUsage > 0: instance is memory-starved, scale up
- Optimize queries to reduce working set size
- Consider partitioning large tables to reduce scan scope

## Safety Ratings
- GREEN: CloudWatch FreeableMemory/SwapUsage metrics, SHOW GLOBAL STATUS, SHOW VARIABLES — read-only inspection
- YELLOW: modify-db-parameter-group (innodb_buffer_pool_size), modify-db-instance (scale up) — recoverable but may require reboot
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "FreeableMemory critically low with swap usage increasing"
- "Fix requires parameter group change that needs reboot"
- "Fix requires instance class change (causes downtime)"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, buffer pool configuration
- MEDIUM: CloudWatch memory metrics, buffer pool statistics, table sizes

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix memory issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest setting innodb_buffer_pool_size above {DBInstanceClassMemory*3/4}"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes (innodb_buffer_pool_size) and reboot if needed"
- "If instance scale-up causes issues, scale back down after confirming"

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
