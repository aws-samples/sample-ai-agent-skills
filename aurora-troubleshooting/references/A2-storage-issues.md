---
title: "A2 — Aurora Storage Issues"
description: "Diagnose Aurora storage volume issues including auto-scaling and space management"
status: active
severity: HIGH
triggers:
  - "storage full"
  - "VolumeBytesUsed"
  - "storage growing"
  - "cannot shrink storage"
  - "storage auto-scaling"
  - "128 TiB limit"
owner: devops-agent
objective: "Identify and resolve Aurora storage issues"
context: "Aurora uses a shared distributed storage volume that auto-scales up to 128 TiB in 10 GiB increments. Storage never shrinks — freed space is reused internally. The I/O model (Standard vs I/O-Optimized) affects cost but not storage behavior."
---

## Phase 1 — Triage

MUST:
- Check current storage usage:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeBytesUsed \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> \
    --start-time <start> --end-time <end> --period 3600 --statistics Average
  ```
- Check storage growth trend over time (use longer period):
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeBytesUsed \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> \
    --start-time <30-days-ago> --end-time <now> --period 86400 --statistics Average
  ```
- Check cluster configuration: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].{Engine:Engine,StorageEncrypted:StorageEncrypted,IOOptimized:StorageType}'`

SHOULD:
- Check I/O metrics to understand workload pattern:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeReadIOPs ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeWriteIOPs ...
  ```
- For Aurora MySQL — check table sizes:
  ```sql
  SELECT table_schema, table_name,
         ROUND(data_length/1024/1024, 2) AS data_mb,
         ROUND(index_length/1024/1024, 2) AS index_mb
  FROM information_schema.tables
  ORDER BY data_length + index_length DESC LIMIT 20;
  ```
- For Aurora PostgreSQL — check table sizes:
  ```sql
  SELECT schemaname, tablename,
         pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size
  FROM pg_tables
  WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
  ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC LIMIT 20;
  ```

MAY:
- Check for bloat (PostgreSQL): `SELECT schemaname, relname, n_dead_tup FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;`
- Check binary log usage (MySQL with binlog enabled): `SHOW BINARY LOGS;`

## Phase 2 — Remediate

MUST:
- Understand that Aurora storage NEVER shrinks. Deleting data frees space for internal reuse only.
- To reclaim storage: create a new cluster from a snapshot, or perform logical dump and restore to a new cluster
- For approaching 128 TiB limit: archive old data, partition large tables, or split into multiple clusters

SHOULD:
- For PostgreSQL bloat: run `VACUUM FULL` on bloated tables (causes table lock)
- For MySQL: run `OPTIMIZE TABLE` on fragmented tables (causes table rebuild)
- Review I/O-Optimized vs Standard pricing if I/O costs are high
- Implement data lifecycle policies to prevent unbounded growth

MAY:
- Consider Aurora cloning for testing storage reclamation strategies
- Evaluate partitioning strategies for large tables

## Common Issues

- symptoms: "Storage keeps growing even after deleting data"
  diagnosis: "Aurora storage never shrinks. Freed space is reused internally."
  resolution: "Create a new cluster from a snapshot to reclaim storage."

- symptoms: "High VolumeBytesUsed with I/O-Optimized"
  diagnosis: "I/O-Optimized does not affect storage behavior, only I/O pricing."
  resolution: "Storage management is the same regardless of I/O pricing model."

- symptoms: "Approaching 128 TiB storage limit"
  diagnosis: "Aurora storage maximum is 128 TiB per cluster."
  resolution: "Archive data, split into multiple clusters, or create a new cluster from snapshot."

## Safety Ratings
- GREEN: describe-db-clusters, CloudWatch VolumeBytesUsed/VolumeReadIOPs/VolumeWriteIOPs metrics, SELECT queries on information_schema/pg_tables — read-only storage inspection
- YELLOW: OPTIMIZE TABLE, VACUUM FULL — recoverable but may cause table locks and performance impact
- RED: delete-db-cluster, dropping large tables without backup — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires VACUUM FULL or OPTIMIZE TABLE on large production tables (causes locks)"
- "Storage approaching 128 TiB limit requiring architectural changes"
- "Fix involves creating new cluster from snapshot to reclaim storage"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings used to run SQL queries
- HIGH: query results from information_schema.tables or pg_tables (reveal schema structure)
- MEDIUM: CloudWatch storage metrics, I/O metrics, backup status

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix storage issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest running VACUUM FULL or OPTIMIZE TABLE on large production tables without confirming maintenance window"
- "NEVER suggest modifying production parameter groups without testing"

## Phase 3 — Rollback
- "Restore from snapshot if storage reclamation via new cluster causes issues"
- "If VACUUM FULL or OPTIMIZE TABLE causes performance degradation, wait for completion — do NOT kill the process"
- "Revert parameter group changes and reboot if needed"

## Output Format

```yaml
root_cause: "storage_issue — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<VolumeBytesUsed metrics>"
  - type: table_sizes
    content: "<largest tables and sizes>"
severity: HIGH
mitigation:
  immediate: "Address immediate storage concern"
  long_term: "Implement data lifecycle management and monitoring"
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
