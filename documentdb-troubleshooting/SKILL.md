---
name: documentdb-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon DocumentDB problems
  by analyzing cluster configurations, instance health, query performance,
  index management, connectivity, replication, and following structured runbooks.
  Activate when: cluster creation failures, instance issues, storage problems,
  query performance degradation, index management, profiler analysis,
  connection failures, TLS configuration, connection pooling, read replica
  lag, global cluster issues, MongoDB compatibility differences, driver
  problems, change stream configuration, consumer issues, snapshot failures,
  PITR, authentication errors, encryption issues, or the user says something
  is wrong with DocumentDB without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with DocumentDB (docdb), CloudWatch,
  CloudTrail, EC2, KMS, IAM, and optionally mongo shell for direct
  cluster diagnostics.
---

# DocumentDB Diagnostics

## When to use

Any Amazon DocumentDB investigation where the console alone is insufficient — cluster health analysis, query performance debugging, connectivity troubleshooting, replication monitoring, MongoDB compatibility issues, change stream problems, backup/restore, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws docdb describe-db-clusters
aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>
aws docdb describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>
aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name CPUUtilization --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name FreeableMemory --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name DatabaseConnections --dimensions Name=DBClusterIdentifier,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws docdb describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group>
aws docdb describe-events --source-type db-cluster --duration 1440
aws docdb describe-events --source-type db-instance --duration 1440
aws docdb describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>
mongo --tls --host <cluster-endpoint> --username <user> --password <pass> --eval "db.serverStatus()"
mongo --tls --host <cluster-endpoint> --username <user> --password <pass> --eval "db.currentOp()"
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rds.amazonaws.com --max-results 20
aws docdb describe-db-cluster-snapshots --db-cluster-identifier <cluster-id>
aws docdb describe-global-clusters
aws cloudwatch get-metric-statistics --namespace AWS/DocDB --metric-name ReadReplicaLag --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum
aws kms describe-key --key-id <key-id>
aws ec2 describe-security-groups --group-ids <sg-id>
```

Read `references/documentdb-guardrails.md` before concluding on any DocumentDB issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-db-clusters` | Cluster configuration and status |
| `describe-db-instances` | Instance details and health |
| `describe-db-cluster-parameters` | Parameter group settings |
| `describe-events` | Recent cluster and instance events |
| `describe-db-cluster-endpoints` | Cluster endpoints configuration |
| `describe-db-cluster-snapshots` | Snapshot details |
| `describe-global-clusters` | Global cluster configuration |
| `mongo serverStatus()` | Real-time server statistics |
| `mongo currentOp()` | Active operations |
| `CloudWatch DocDB metrics` | Performance and capacity metrics |

## Gotchas: Amazon DocumentDB

- DocumentDB is NOT MongoDB. It is API-compatible with MongoDB 3.6, 4.0, and 5.0 but uses a different storage engine (Aurora-based). Not all MongoDB features are supported. Always check the DocumentDB documentation for supported operations, not MongoDB documentation.
- TLS is required by default. All connections must use TLS. The rds-combined-ca-bundle.pem certificate must be specified in the connection string. Disabling TLS requires modifying the cluster parameter group (tls=disabled), which is NOT recommended for production.
- No $lookup across collections in older versions. DocumentDB 4.0 added limited $lookup support. DocumentDB 5.0 has broader $lookup support. Always verify aggregation pipeline stage support for your engine version.
- Change streams have 24-hour retention. Change stream events are retained for 24 hours (configurable up to 7 days with change_stream_log_retention_duration parameter). Consumers must process events within the retention window or they are lost. Change streams require a resume token for reliable processing.
- Elastic clusters use sharding (different from standard). Elastic clusters distribute data across shards using a shard key. Standard clusters use a single-writer, multi-reader architecture. Elastic clusters have different API endpoints and connection patterns. Not all features available in standard clusters work in elastic clusters.
- Parameter groups differ from MongoDB config. DocumentDB uses cluster parameter groups (similar to RDS), not MongoDB configuration files. Parameters like profiler, TTL monitor, and audit logging are controlled via parameter groups. Some MongoDB parameters have no DocumentDB equivalent.
- No direct mongodump/mongorestore. DocumentDB does not support mongodump/mongorestore for backup. Use AWS native snapshots, PITR, or AWS DMS for data migration. For logical exports, use mongoexport/mongoimport or custom scripts.
- Index builds are foreground by default. Index creation blocks writes on the collection in DocumentDB (unlike MongoDB 4.2+ which has background index builds). Plan index creation during maintenance windows. Monitor index build progress with currentOp().

## Anti-hallucination rules

1. Always cite specific cluster identifiers, instance IDs, CloudWatch metrics, or mongo shell output as evidence.
2. DocumentDB is NOT MongoDB. Never reference MongoDB-specific features without verifying DocumentDB support.
3. TLS is required by default. Never suggest connecting without TLS unless the parameter is explicitly disabled.
4. Change streams have limited retention (24h default). Never assume unlimited retention.
5. Index builds block writes. Never suggest building indexes on production collections without planning for write impact.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A3 | Creation failures, instance issues, storage |
| B — Performance | B1-B3 | Query performance, index management, profiler |
| C — Connectivity | C1-C3 | Connection failures, TLS, connection pooling |
| D — Replication | D1-D2 | Read replicas, global clusters |
| E — Compatibility | E1-E2 | MongoDB differences, driver issues |
| F — Change Streams | F1-F2 | Stream configuration, consumer issues |
| G — Backup | G1-G2 | Snapshot failures, PITR |
| H — Security | H1-H2 | Authentication, encryption |
| Z — Catch-All | Z1 | General troubleshooting |
