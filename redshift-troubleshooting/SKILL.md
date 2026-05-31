---
name: redshift-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Redshift problems
  by analyzing query performance, WLM queue configuration, disk space,
  node failures, COPY/UNLOAD errors, Spectrum queries, data sharing,
  snapshot/restore, cluster resize, and concurrency scaling. Covers both
  Redshift provisioned clusters and Redshift Serverless. Activate when:
  slow queries, WLM queue wait times, lock contention, query compilation
  delays, disk full, data skew, VACUUM issues, COPY failures, UNLOAD
  errors, Spectrum external table errors, node failures, resize operations,
  maintenance window issues, concurrency scaling problems, IAM/VPC
  security, encryption, data sharing producer/consumer issues,
  cross-account sharing, Serverless RPU configuration, endpoint access,
  snapshot failures, restore issues, or the user says something is wrong
  with Redshift without naming specific symptoms.
compatibility: >
  Requires AWS CLI with redshift, redshift-data, redshift-serverless,
  cloudwatch, and cloudtrail permissions. Optional: psql client for
  direct cluster queries, S3 permissions for COPY/UNLOAD debugging.
---

# Redshift Diagnostics

## When to use

Any Redshift investigation where the console alone is insufficient — query performance, WLM tuning, storage management, data loading, cluster operations, Spectrum, data sharing, Serverless configuration, or security issues.

## Investigation workflow

### Step 1 — Collect and triage

```
# List provisioned clusters
aws redshift describe-clusters --query 'Clusters[*].{id:ClusterIdentifier,status:ClusterStatus,nodes:NumberOfNodes,type:NodeType}'

# List Serverless workgroups
aws redshift-serverless list-workgroups --query 'workgroups[*].{name:workgroupName,status:status,baseRPU:baseCapacity}'
version: "1.0.0"
last_updated: "2025-04-12"

# Check cluster health via CloudWatch
aws cloudwatch get-metric-statistics --namespace AWS/Redshift \
  --metric-name HealthStatus --dimensions Name=ClusterIdentifier,Value=<cluster-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Average

# Check disk space usage
aws cloudwatch get-metric-statistics --namespace AWS/Redshift \
  --metric-name PercentageDiskSpaceUsed --dimensions Name=ClusterIdentifier,Value=<cluster-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Maximum

# Check CPU utilization
aws cloudwatch get-metric-statistics --namespace AWS/Redshift \
  --metric-name CPUUtilization --dimensions Name=ClusterIdentifier,Value=<cluster-id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 --statistics Average

# Run a query via Redshift Data API
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT query, elapsed, substring FROM svl_qlog ORDER BY starttime DESC LIMIT 20"
```

Triage returns:
- Cluster/workgroup state, node count, node type
- Disk space percentage, CPU utilization, health status
- Recent query execution times and patterns

If the cluster status is not "available", that IS the root cause domain. Don't chase query or storage symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Check WLM queue configuration
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM stv_wlm_service_class_config WHERE service_class >= 6"

# Check running and queued queries
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM stv_wlm_query_state"

# Check lock contention
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM svv_transactions WHERE lockable_object_type IS NOT NULL AND granted = 'f'"

# Check table skew and distribution
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT schema, \"table\", diststyle, skew_rows FROM svv_table_info WHERE skew_rows > 4.0 ORDER BY skew_rows DESC"

# Check COPY errors
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM stl_load_errors ORDER BY starttime DESC LIMIT 20"

# CloudTrail for Redshift API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=redshift.amazonaws.com --max-results 20
```

Read `references/redshift-guardrails.md` before concluding on any Redshift issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# Query execution plan analysis
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM svl_query_summary WHERE query = <query-id> ORDER BY stm, seg, step"

# Check VACUUM status
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM svv_table_info WHERE unsorted > 5 OR empty > 20 ORDER BY unsorted DESC"

# Check Spectrum query details
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM svl_s3query_summary ORDER BY starttime DESC LIMIT 10"

# Check concurrency scaling activity
aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> \
  --sql "SELECT * FROM stl_concurrency_scaling_usage ORDER BY starttime DESC LIMIT 20"

# Data sharing status
aws redshift describe-data-shares --data-share-arn <share-arn>

# Snapshot status
aws redshift describe-cluster-snapshots --cluster-identifier <cluster-id> --query 'Snapshots[*].{id:SnapshotIdentifier,status:Status,type:SnapshotType,time:SnapshotCreateTime}'
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws redshift describe-clusters` | Cluster status, node type, node count |
| `aws redshift-serverless list-workgroups` | Serverless workgroup status and RPU config |
| `aws redshift-data execute-statement` | Run SQL queries via Data API |
| `aws cloudwatch get-metric-statistics` | Redshift metrics: CPU, disk, query duration |
| `aws redshift describe-cluster-snapshots` | Snapshot status and history |
| `aws redshift describe-data-shares` | Data sharing configuration |
| `aws redshift describe-resize` | Resize operation progress |
| `aws cloudtrail lookup-events` | Redshift API call history |
| `psql` | Direct SQL access for system tables |
| `stl_load_errors` | COPY command error details |
| `svv_table_info` | Table distribution, skew, sort key stats |
| `stv_wlm_query_state` | WLM queue state and running queries |

## Gotchas: Redshift

These are the mistakes commonly made during Redshift troubleshooting.

- Leader node vs compute nodes — the leader node handles SQL parsing, query planning, and final result aggregation. Compute nodes store data and execute query fragments. Queries that run entirely on the leader (e.g., SELECT 1, functions with no table reference) don't use compute nodes at all. System tables like SVL_QLOG are on the leader.
- WLM queues control concurrency — the default WLM configuration allows 5 concurrent queries per queue. Queries beyond the concurrency limit wait in the queue. Auto WLM is recommended for most workloads as it dynamically adjusts concurrency and memory allocation.
- COPY is the fastest way to load data — not INSERT. COPY loads from S3, DynamoDB, or remote hosts in parallel across all compute nodes. Single-row INSERTs are extremely slow because each one is a separate transaction. Always use COPY for bulk loading.
- Distribution keys determine data placement across nodes — choosing the wrong distribution key causes data skew (one node has much more data than others) and excessive network redistribution during joins. Check svv_table_info for skew_rows > 4.0.
- Sort keys affect scan performance — compound sort keys are useful for queries that filter on leading columns. Interleaved sort keys help when queries filter on different columns but have higher VACUUM overhead. Unsorted data degrades query performance.
- VACUUM is needed after deletes and updates — Redshift does not automatically reclaim space from deleted rows for all operations. Auto-VACUUM handles DELETE reclamation but manual VACUUM SORT may still be needed for sort key maintenance. Check the "unsorted" percentage in svv_table_info.
- Redshift Spectrum queries S3 directly via external tables — Spectrum uses compute nodes to scan S3 data. It does NOT load data into Redshift storage. Performance depends on S3 file format (Parquet/ORC best), file size (100MB-1GB optimal), and partition pruning.
- Concurrency scaling adds transient clusters for burst read workloads — it only handles read queries, not writes. Concurrency scaling clusters are billed per-second. It activates automatically when WLM queues have waiting queries.
- Serverless has RPU-based pricing — Redshift Serverless uses Redshift Processing Units (RPUs) instead of fixed node types. Base capacity is configurable (min 8 RPU). You pay for compute time in RPU-seconds. There are no nodes to manage.
- Cross-database queries have limitations — you can query across databases in the same cluster using three-part notation (database.schema.table), but you cannot write across databases. External schemas (Spectrum) cannot be queried cross-database.
- Maintenance windows can cause brief unavailability — during maintenance, the cluster may be restarted. Schedule maintenance windows during low-traffic periods. Deferred maintenance is available but has limits.
- Resize operations — classic resize creates a new cluster and migrates data (hours). Elastic resize adds/removes nodes in minutes but has node type constraints. RA3 nodes with managed storage make elastic resize much faster.

## Anti-hallucination rules

1. Always cite specific system table output (STL_, SVL_, SVV_, STV_), CloudWatch metrics, or AWS CLI output as evidence.
2. Never suggest SSH access to Redshift nodes — Redshift is fully managed with no OS-level access.
3. Never recommend INSERT for bulk loading — always use COPY from S3.
4. Never claim VACUUM is unnecessary — it is required for sort key maintenance and space reclamation after deletes.
5. Never suggest changing distribution style without analyzing join patterns and data skew first.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Query Performance | A1–A4 | Slow queries, queue wait times, lock contention, query compilation |
| B — Storage | B1–B3 | Disk full, data skew, VACUUM issues |
| C — Data Loading | C1–C3 | COPY failures, UNLOAD errors, Spectrum external tables |
| D — Cluster Operations | D1–D3 | Node failures, resize operations, maintenance |
| E — WLM | E1–E2 | Queue configuration, concurrency scaling |
| F — Security | F1–F2 | IAM/VPC configuration, encryption |
| G — Data Sharing | G1–G2 | Producer/consumer issues, cross-account sharing |
| H — Serverless | H1–H2 | RPU configuration, endpoint access |
| I — Snapshots | I1–I2 | Backup failures, restore issues |
| Z — Catch-All | Z1 | General Redshift troubleshooting |
