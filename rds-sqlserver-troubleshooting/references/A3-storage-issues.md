---
title: "A3 — RDS SQL Server Storage Issues"
description: "Diagnose storage capacity, performance, and configuration problems"
status: active
severity: HIGH
triggers:
  - "storage full"
  - "FreeStorageSpace"
  - "storage autoscaling"
  - "IOPS"
  - "storage type"
  - "gp2"
  - "gp3"
  - "io1"
  - "disk full"
  - "could not allocate space"
owner: devops-agent
objective: "Resolve RDS SQL Server storage capacity, performance, and type issues"
context: "RDS SQL Server uses EBS storage. Storage types: gp2, gp3, io1, io2. Maximum 16 TiB. Storage autoscaling available. Storage can only be increased, never decreased. Storage modifications have a 6-hour cooldown."
---

## Phase 1 — Triage

MUST:
- Check current storage: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Storage:AllocatedStorage,Type:StorageType,IOPS:Iops,Throughput:StorageThroughput}'`
- Check free storage: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check storage autoscaling: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].MaxAllocatedStorage'`
- Check I/O metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS ...`
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS ...`
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...`
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...`

SHOULD:
- Check database file sizes via T-SQL:
  ```sql
  SELECT DB_NAME(database_id) AS db_name, name, type_desc,
         size*8/1024 AS size_mb,
         CAST(FILEPROPERTY(name,'SpaceUsed') AS INT)*8/1024 AS used_mb
  FROM sys.master_files ORDER BY size DESC;
  ```
- Check for large tables:
  ```sql
  SELECT TOP 20 s.name + '.' + t.name AS table_name,
         SUM(p.rows) AS row_count,
         SUM(a.total_pages)*8/1024 AS total_mb
  FROM sys.tables t
  JOIN sys.schemas s ON t.schema_id = s.schema_id
  JOIN sys.partitions p ON t.object_id = p.object_id AND p.index_id IN (0,1)
  JOIN sys.allocation_units a ON p.partition_id = a.container_id
  GROUP BY s.name, t.name ORDER BY total_mb DESC;
  ```
- Check pending storage modifications: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PendingModifiedValues'`

MAY:
- Check for storage optimization opportunities (unused indexes, log file bloat)
- Review Enhanced Monitoring for disk queue depth

## Phase 2 — Remediate

MUST:
- For storage full: increase allocated storage (cannot decrease)
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --allocated-storage <new-size-gb> \
    --apply-immediately
  ```
- For I/O bottleneck: consider storage type change (gp2→gp3 or io1/io2 for provisioned IOPS)
- Enable storage autoscaling if not set:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --max-allocated-storage <max-gb>
  ```

SHOULD:
- Shrink transaction log if bloated:
  ```sql
  -- Check log space usage
  DBCC SQLPERF(LOGSPACE);
  -- Shrink log file (after ensuring backup)
  USE [MyDatabase];
  DBCC SHRINKFILE(N'MyDatabase_log', 1024);  -- Target size in MB
  ```
- Wait 6 hours between storage modifications (cooldown period)
- For gp3: configure IOPS and throughput independently of storage size

MAY:
- Migrate to io1/io2 for consistent high IOPS workloads
- Implement database file size monitoring alerts

## Common Issues

- symptoms: "Could not allocate space for object in database"
  diagnosis: "Database data file full. RDS storage may have space but database file needs growing."
  resolution: "Check database file autogrowth settings. If RDS storage is also full, increase allocated storage first."

- symptoms: "Storage modification stuck — 6-hour cooldown"
  diagnosis: "RDS enforces a 6-hour cooldown between storage modifications."
  resolution: "Wait for cooldown. Plan storage increases with sufficient headroom."

- symptoms: "High I/O latency on gp2"
  diagnosis: "gp2 IOPS scale with volume size (3 IOPS/GB, burst to 3000). Small volumes have low baseline IOPS."
  resolution: "Switch to gp3 (3000 baseline IOPS regardless of size) or io1/io2 for provisioned IOPS."

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch FreeStorageSpace/ReadIOPS/WriteIOPS metrics, SELECT from sys.master_files/sys.tables — read-only inspection
- YELLOW: modify-db-instance (storage increase/type change), DBCC SHRINKFILE — recoverable changes
- RED: delete-db-instance, dropping databases — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Storage full causing application errors"
- "Fix requires storage type change (6-hour cooldown)"
- "Fix involves shrinking transaction log on production"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, database file details (reveal schema structure)
- MEDIUM: CloudWatch storage metrics, IOPS configuration, table sizes

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix storage issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest DBCC SHRINKDATABASE on production without understanding fragmentation impact"

## Phase 3 — Rollback
- "Restore from snapshot if storage configuration change causes issues"
- "Storage increases cannot be reversed — plan increases with sufficient headroom"
- "Wait 6 hours between storage modifications (cooldown period)"

## Output Format

```yaml
root_cause: "storage — <specific_cause>"
evidence:
  - type: storage_config
    content: "<current storage type, size, IOPS>"
  - type: metrics
    content: "<FreeStorageSpace, IOPS, latency metrics>"
severity: HIGH
mitigation:
  immediate: "Increase storage or change storage type"
  long_term: "Enable storage autoscaling and implement file size monitoring"
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
