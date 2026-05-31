---
title: "B3 — I/O Bottleneck"
description: "Diagnose I/O performance issues on RDS Oracle"
status: active
severity: HIGH
triggers:
  - "I/O"
  - "IOPS"
  - "latency"
  - "ReadLatency"
  - "WriteLatency"
  - "db file sequential read"
  - "db file scattered read"
  - "log file sync"
  - "slow disk"
owner: devops-agent
objective: "Identify and resolve I/O bottlenecks affecting RDS Oracle performance"
context: "RDS Oracle uses EBS storage. I/O performance depends on storage type (gp2/gp3/io1/io2), volume size, and instance class EBS bandwidth. Oracle I/O waits (db file sequential read, db file scattered read, log file sync) are the primary indicators."
---

## Phase 1 — Triage

MUST:
- Check I/O metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DiskQueueDepth ...
  ```
- Check storage configuration: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Type:StorageType,Size:AllocatedStorage,IOPS:Iops}'`
- Check I/O wait events:
  ```sql
  SELECT event, total_waits, time_waited/100 AS time_sec, average_wait/100 AS avg_sec
  FROM v$system_event
  WHERE event IN ('db file sequential read','db file scattered read',
                  'log file sync','log file parallel write','direct path read')
  ORDER BY time_waited DESC;
  ```

SHOULD:
- Check top I/O SQL:
  ```sql
  SELECT sql_id, disk_reads, buffer_gets, executions,
         disk_reads/NULLIF(executions,0) AS reads_per_exec
  FROM v$sql ORDER BY disk_reads DESC FETCH FIRST 10 ROWS ONLY;
  ```
- Check for full table scans causing scattered reads:
  ```sql
  SELECT sql_id, object_name, operation, options
  FROM v$sql_plan WHERE options = 'FULL' AND object_owner NOT IN ('SYS','SYSTEM');
  ```
- Check buffer cache hit ratio:
  ```sql
  SELECT 1 - (physical_reads / (db_block_gets + consistent_gets)) AS hit_ratio
  FROM (SELECT value AS physical_reads FROM v$sysstat WHERE name = 'physical reads'),
       (SELECT value AS db_block_gets FROM v$sysstat WHERE name = 'db block gets'),
       (SELECT value AS consistent_gets FROM v$sysstat WHERE name = 'consistent gets');
  ```

MAY:
- Check Enhanced Monitoring for OS-level I/O stats
- Review I/O calibration: `SELECT * FROM v$io_calibration_status;`

## Phase 2 — Remediate

MUST:
- For IOPS limit: upgrade storage type (gp2→gp3 or io1/io2) or increase provisioned IOPS
- For high read latency: optimize SQL to reduce disk reads, add indexes
- For log file sync: check commit frequency, batch commits where possible

SHOULD:
- Increase buffer cache (db_cache_size within SGA) to reduce physical reads
- Gather fresh optimizer statistics: `EXEC DBMS_STATS.GATHER_SCHEMA_STATS('<owner>');`
- Consider gp3 for independent IOPS/throughput configuration

MAY:
- Enable asynchronous I/O: check `disk_asynch_io` parameter
- Use parallel query for large scan operations (EE only)

## Common Issues

- symptoms: "High 'db file sequential read' waits"
  diagnosis: "Single-block reads (index lookups) hitting disk. Buffer cache miss or undersized cache."
  resolution: "Increase buffer cache. Add indexes to reduce random I/O. Increase IOPS."

- symptoms: "High 'log file sync' waits"
  diagnosis: "Commit-heavy workload. Each COMMIT waits for redo log write to complete."
  resolution: "Batch commits. Reduce commit frequency. Upgrade to io1/io2 for lower write latency."

- symptoms: "DiskQueueDepth consistently > 1"
  diagnosis: "I/O requests queuing. Storage throughput saturated."
  resolution: "Increase provisioned IOPS or switch to higher-performance storage type."

## Safety Ratings
- GREEN: CloudWatch ReadIOPS/WriteIOPS/ReadLatency/WriteLatency/DiskQueueDepth metrics, SELECT from v$system_event/v$sql — read-only inspection
- YELLOW: modify-db-instance (storage type/IOPS change), EXEC DBMS_STATS.GATHER_SCHEMA_STATS — recoverable changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "I/O latency causing application timeouts"
- "Fix requires storage type change (6-hour cooldown between modifications)"
- "Fix requires parameter group change that needs reboot"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, SQL text from v$sql (may contain sensitive data)
- HIGH: query results from v$system_event (reveal I/O patterns and workload characteristics)
- MEDIUM: CloudWatch I/O metrics, storage configuration, buffer cache statistics

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix I/O issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "Restore from snapshot if storage configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "Wait 6 hours between storage modifications (cooldown period)"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "io_bottleneck — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<IOPS, latency, queue depth metrics>"
  - type: wait_events
    content: "<I/O wait events from v$system_event>"
  - type: top_sql
    content: "<top I/O SQL from v$sql>"
severity: HIGH
mitigation:
  immediate: "Optimize top I/O SQL or increase IOPS"
  long_term: "Right-size storage type and implement I/O monitoring"
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
