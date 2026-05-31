---
title: "B3 — Aurora I/O Issues"
description: "Diagnose I/O performance issues specific to the Aurora storage model"
status: active
severity: HIGH
triggers:
  - "I/O"
  - "IOPS"
  - "VolumeReadIOPs"
  - "VolumeWriteIOPs"
  - "ReadLatency"
  - "WriteLatency"
  - "I/O-Optimized"
  - "io/aurora"
owner: devops-agent
objective: "Identify and resolve I/O performance issues in Aurora's distributed storage model"
context: "Aurora uses a distributed storage volume with 6 copies across 3 AZs. I/O behavior differs from standard RDS (EBS-based). Aurora has two pricing models: Standard (per-I/O charges) and I/O-Optimized (no I/O charges, higher instance cost). Aurora-specific I/O wait events include io/aurora_redo_log_flush and io/aurora_respond_to_client."
---

## Phase 1 — Triage

MUST:
- Check Aurora-specific I/O metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeReadIOPs \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Sum
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeWriteIOPs ...
  ```
- Check instance-level I/O latency:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...
  ```
- Check I/O wait events via Performance Insights:
  ```
  aws pi get-resource-metrics --service-type RDS --identifier db-<resource-id> \
    --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.wait_event"}}]' \
    --start-time <start> --end-time <end> --period-in-seconds 300
  ```
- Check storage type: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].StorageType'`

SHOULD:
- For Aurora MySQL — check I/O-related waits:
  ```sql
  SELECT event_name, count_star, sum_timer_wait/1000000000 AS total_wait_ms
  FROM performance_schema.events_waits_summary_global_by_event_name
  WHERE event_name LIKE 'io/aurora%' ORDER BY sum_timer_wait DESC;
  ```
- For Aurora PostgreSQL — check I/O waits:
  ```sql
  SELECT wait_event_type, wait_event, count(*)
  FROM pg_stat_activity WHERE wait_event_type = 'IO'
  GROUP BY wait_event_type, wait_event ORDER BY count DESC;
  ```
- Check for full table scans driving high I/O:
  - Aurora MySQL: `SHOW STATUS LIKE 'Select_scan';`
  - Aurora PostgreSQL: `SELECT schemaname, relname, seq_scan, seq_tup_read FROM pg_stat_user_tables ORDER BY seq_tup_read DESC LIMIT 10;`

MAY:
- Compare I/O costs between Standard and I/O-Optimized pricing
- Check if parallel query (MySQL only) could reduce I/O for analytical queries

## Phase 2 — Remediate

MUST:
- Optimize queries causing excessive I/O (add indexes, reduce full table scans)
- For write-heavy workloads with high redo log flush waits: batch commits, reduce write frequency
- Scale up instance class if I/O throughput is limited by instance network bandwidth

SHOULD:
- Evaluate switching to I/O-Optimized if I/O charges exceed 25% of total Aurora cost
- For Aurora MySQL: consider parallel query for analytical workloads
- Optimize buffer pool / shared_buffers to reduce physical reads

MAY:
- Use Aurora cloning to test I/O optimization strategies
- Implement read/write splitting to distribute I/O across instances

## Common Issues

- symptoms: "High VolumeWriteIOPs with io/aurora_redo_log_flush waits"
  diagnosis: "Write-heavy workload. Each commit flushes redo log to Aurora storage."
  resolution: "Batch commits. Use multi-row inserts. Reduce commit frequency."

- symptoms: "High VolumeReadIOPs with full table scans"
  diagnosis: "Queries scanning entire tables instead of using indexes."
  resolution: "Add appropriate indexes. Optimize query WHERE clauses."

- symptoms: "I/O costs are high on Standard pricing"
  diagnosis: "I/O-heavy workload on Standard pricing model."
  resolution: "Evaluate I/O-Optimized pricing. Can switch once every 30 days."

## Safety Ratings
- GREEN: describe-db-clusters, CloudWatch VolumeReadIOPs/VolumeWriteIOPs/ReadLatency/WriteLatency metrics, Performance Insights, SELECT from performance_schema/pg_stat_activity — read-only inspection
- YELLOW: modify-db-instance (scale up), switching I/O-Optimized pricing — recoverable changes
- RED: force-failover, delete-db-instance — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "I/O latency causing application timeouts"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, query text from wait event analysis
- HIGH: query results from performance_schema.events_waits, pg_stat_activity (contain SQL text and I/O patterns)
- MEDIUM: CloudWatch I/O metrics, storage type configuration, Performance Insights data

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix I/O issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "I/O-Optimized pricing can only be switched once every 30 days — plan accordingly"

## Output Format

```yaml
root_cause: "io_issue — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<VolumeReadIOPs, VolumeWriteIOPs, latency metrics>"
  - type: wait_events
    content: "<I/O wait events>"
  - type: top_sql
    content: "<top I/O-consuming queries>"
severity: HIGH
mitigation:
  immediate: "Optimize top I/O queries or evaluate I/O-Optimized pricing"
  long_term: "Implement I/O monitoring and query optimization practices"
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
