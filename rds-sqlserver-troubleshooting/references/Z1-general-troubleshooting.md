---
title: "Z1 — General RDS SQL Server Troubleshooting (Catch-All)"
description: "Fallback SOP for RDS SQL Server issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown RDS SQL Server issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws rds describe-db-instances --db-instance-identifier <instance-id>`
- Check recent events: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Check key CloudWatch metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
  ```
- Check error log: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG --number-of-lines 500`
- Check parameter group status: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBParameterGroups'`

SHOULD:
- Check SQL Server-level health:
  ```sql
  SELECT name, state_desc, recovery_model_desc FROM sys.databases;

  SELECT TOP 20 wait_type, wait_time_ms/1000.0 AS wait_sec, waiting_tasks_count
  FROM sys.dm_os_wait_stats
  WHERE wait_type NOT IN ('SLEEP_TASK','BROKER_TO_FLUSH','SQLTRACE_BUFFER_FLUSH',
    'CLR_AUTO_EVENT','CLR_MANUAL_EVENT','LAZYWRITER_SLEEP','CHECKPOINT_QUEUE',
    'WAITFOR','XE_TIMER_EVENT','FT_IFTS_SCHEDULER_IDLE_WAIT','BROKER_TASK_STOP',
    'DIRTY_PAGE_POLL','HADR_FILESTREAM_IOMGR_IOCOMPLETION','SP_SERVER_DIAGNOSTICS_SLEEP')
  ORDER BY wait_time_ms DESC;

  SELECT COUNT(*) AS total_connections,
         SUM(CASE WHEN status = 'running' THEN 1 ELSE 0 END) AS active,
         SUM(CASE WHEN status = 'sleeping' THEN 1 ELSE 0 END) AS sleeping
  FROM sys.dm_exec_sessions WHERE is_user_process = 1;

  SELECT DB_NAME(database_id) AS db_name, name, type_desc,
         size*8/1024 AS size_mb
  FROM sys.master_files WHERE database_id > 4 ORDER BY size DESC;
  ```
- Check pending modifications: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PendingModifiedValues'`
- Check pending maintenance: `aws rds describe-pending-maintenance-actions --resource-identifier <instance-arn>`
- Check option group: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].OptionGroupMemberships'`

MAY:
- Check Performance Insights for DB load trends
- Check Enhanced Monitoring for OS-level metrics
- Review CloudTrail for recent configuration changes

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Instance launch/config issues → Instance (A1-A3)
- CPU, memory, I/O, tempdb → Performance (B1-B4)
- Connection failures, Windows Auth, SSL → Connectivity (C1-C3)
- Parameter group, config, memory/workers → Parameters (D1-D3)
- Native backup, snapshot, PITR → Backup & Recovery (E1-E3)
- Read replica, Multi-AZ → Replication (F1-F2)
- Native restore, DMS, collation → Migration (G1-G3)
- TDE, audit, login/user → Security (H1-H3)
- Upgrades, patching → Maintenance (I1-I2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all instance configurations systematically
- Review CloudTrail for recent API calls affecting the instance
- Check for errors in SQL Server error log and agent log
- Verify network connectivity and security group rules

SHOULD:
- Check AWS Health Dashboard for RDS service events
- Compare with a known-good instance configuration
- Check for recent changes (parameter group, option group, security group)

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings
- GREEN: describe-db-instances, describe-events, CloudWatch metrics, SELECT from sys.dm_exec_sessions/sys.dm_os_wait_stats/sys.configurations — read-only inspection
- YELLOW: modify-db-instance, modify-db-parameter-group — recoverable configuration changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Root cause cannot be identified — escalate to AWS Support"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Multi-AZ instance"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, error log contents (may contain SQL text and sensitive data)
- HIGH: sys.dm_exec_requests output (contains active SQL text and session details)
- MEDIUM: CloudWatch metrics, instance configuration, pending maintenance actions

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<instance|performance|connectivity|parameters|backup|replication|migration|security|maintenance|unknown>"
investigation_path: "instance status → events → metrics → SQL Server health → <domain_classification>"
evidence:
  - type: instance_config
    content: "<instance configuration summary>"
  - type: cloudwatch
    content: "<key metrics>"
  - type: sqlserver_health
    content: "<SQL Server-level diagnostics>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
