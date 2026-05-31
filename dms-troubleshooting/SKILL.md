---
name: dms-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Database Migration Service
  (DMS) problems by analyzing task failures, source/target endpoint connectivity,
  replication instance sizing, schema conversion, CDC replication lag, data
  validation, LOB handling, and following structured runbooks. Activate when:
  migration task failures, task stuck or not progressing, CDC replication lag,
  source endpoint connectivity issues, source-specific errors, supplemental
  logging not configured, target endpoint errors, data type mapping failures,
  target apply errors, replication instance undersized, storage full on
  replication instance, schema conversion errors, table mapping issues,
  throughput bottlenecks, latency problems, LOB handling failures, data
  validation mismatches, row count discrepancies, IAM role misconfiguration,
  VPC/subnet connectivity problems, or the user says something is wrong
  with DMS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with DMS, IAM, CloudWatch, CloudTrail,
  EC2 (for VPC/subnet), and optionally RDS/Aurora permissions.
  Some operations require database-level access for source/target verification.
---

# AWS DMS Diagnostics

## When to use

Any AWS DMS investigation where the console alone is insufficient — task failures, CDC lag, endpoint connectivity, replication instance issues, schema conversion, data validation, performance tuning, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id>
aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id>
aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id>
aws dms describe-table-statistics --replication-task-arn <task-arn>
```

### Step 2 — Domain deep dive

```
aws dms describe-replication-task-assessment-results --replication-task-arn <task-arn>
aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <endpoint-arn>
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencySource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws dms describe-event-subscriptions
```

### Step 3 — Detailed investigation

```
aws dms describe-replication-task-individual-assessments --filters Name=replication-task-assessment-run-arn,Values=<run-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=dms.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FreeStorageSpace --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average
```

Read `references/dms-guardrails.md` before concluding on any DMS issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-replication-tasks` | Check task status, settings, CDC stats |
| `describe-replication-instances` | Check instance class, storage, status |
| `describe-endpoints` | Check endpoint configuration and connectivity |
| `test-connection` | Test endpoint connectivity from replication instance |
| `describe-table-statistics` | Check per-table load/CDC statistics |
| `describe-replication-task-assessment-results` | Check pre-migration assessments |
| CloudWatch DMS metrics | CDCLatencySource, CDCLatencyTarget, FreeStorageSpace |
| `describe-event-subscriptions` | Check DMS event notifications |

## Gotchas: AWS DMS

- CDC requires supplemental logging on the source. Without it, DMS cannot capture changes. Oracle needs supplemental logging at database or table level. MySQL needs binlog_format=ROW. PostgreSQL needs wal_level=logical.
- LOB columns can drastically slow migration. Full LOB mode reads each LOB individually. Limited LOB mode truncates at max size. Inline LOB mode is fastest but has size limits. Choose the right mode for your data.
- Replication instance storage fills up during large migrations. DMS uses local storage for sorting, caching, and transaction logs. Monitor FreeStorageSpace. Increase storage or instance size proactively.
- Table mappings control what gets migrated. Selection rules choose tables. Transformation rules rename or modify. Incorrect wildcards can include/exclude wrong tables. Test mappings before full migration.
- Data validation runs after full load and during CDC. It compares source and target row by row. Validation adds overhead. Mismatches may be due to ongoing changes, not actual errors.
- Multi-AZ replication instances provide HA but not performance. The standby is for failover only. It does not handle read traffic. Multi-AZ doubles the cost.
- VPC peering or VPN may be needed for on-premises sources. The replication instance must reach both source and target. Security groups, NACLs, and route tables must all allow traffic.
- Task settings JSON controls detailed behavior. BatchApplyEnabled, ParallelLoadThreads, MaxFullLoadSubTasks all affect performance. Default settings are conservative.
- Schema conversion and DMS are separate tools. AWS SCT converts schema; DMS migrates data. Use SCT first for heterogeneous migrations.

## Anti-hallucination rules

1. Always cite specific task ARNs, endpoint IDs, or API responses as evidence.
2. CDC requires source-specific configuration. Never assume it works out of the box.
3. LOB handling mode significantly impacts performance. Never ignore LOB settings.
4. Replication instance class affects throughput. Never assume default is sufficient.
5. Data validation mismatches during CDC may be transient. Never declare failure without re-checking.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Task | A1–A3 | Task failures, task stuck, CDC replication lag |
| B — Source | B1–B3 | Source endpoint connectivity, source-specific errors, supplemental logging |
| C — Target | C1–C3 | Target endpoint errors, data type mapping, target apply errors |
| D — Instance | D1–D2 | Replication instance sizing, storage full |
| E — Schema | E1–E2 | Schema conversion, table mapping |
| F — Performance | F1–F3 | Throughput, latency, LOB handling |
| G — Validation | G1–G2 | Data validation failures, row count mismatch |
| H — Security | H1–H2 | IAM roles, VPC/subnet config |
| Z — Catch-All | Z1 | General troubleshooting |
