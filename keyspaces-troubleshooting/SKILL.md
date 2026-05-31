---
name: keyspaces-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Keyspaces (for Apache Cassandra)
  problems by analyzing table creation, schema management, CQL compatibility,
  read/write performance, connectivity, capacity modes, TTL behavior, backup/restore,
  migration, and following structured runbooks. Activate when: table creation failures,
  schema modification errors, capacity mode issues, read/write throttling, hot partitions,
  query performance degradation, CQL compatibility problems, unsupported Cassandra features,
  connection failures, TLS/SigV4 authentication errors, on-demand vs provisioned confusion,
  auto-scaling misconfiguration, TTL not expiring, TTL configuration errors, PITR restore
  failures, snapshot issues, Cassandra migration problems, CQLReplicator errors, or the
  user says something is wrong with Amazon Keyspaces without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Keyspaces, CloudWatch, IAM, CloudTrail,
  and optionally KMS and Application Auto Scaling permissions. CQL operations
  require cqlsh with SigV4 plugin or service-specific credentials over TLS.
---

# Amazon Keyspaces (for Apache Cassandra) Diagnostics

## When to use

Any Amazon Keyspaces investigation where the console alone is insufficient — table creation failures, schema issues, CQL compatibility, read/write throttling, hot partitions, connectivity problems, TLS/SigV4 authentication, capacity planning, TTL behavior, PITR backup/restore, or Cassandra migration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws keyspaces list-keyspaces
aws keyspaces get-keyspace --keyspace-name <keyspace>
aws keyspaces list-tables --keyspace-name <keyspace>
aws keyspaces get-table --keyspace-name <keyspace> --table-name <table>
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name PerConnectionRequestRateExceeded --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ReadThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name WriteThrottleEvents --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SuccessfulRequestLatency --dimensions Name=TableName,Value=<table> Name=Operation,Value=SELECT --start-time <start> --end-time <end> --period 300 --statistics Average
aws application-autoscaling describe-scalable-targets --service-namespace cassandra --resource-ids keyspace/<keyspace>/table/<table>
aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query '{CapacityMode:CapacitySpecification,TTL:Ttl,PITR:PointInTimeRecovery}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cassandra.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name SystemErrors --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedReadCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Cassandra --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/keyspaces-guardrails.md` before concluding on any Keyspaces issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-keyspace` | Check keyspace existence and configuration |
| `get-table` | Check table schema, capacity mode, TTL, PITR status |
| `list-tables` | List all tables in a keyspace |
| `create-table` | Create a new table (verify schema first) |
| `update-table` | Modify capacity mode, TTL, or PITR settings |
| `restore-table` | Restore table from PITR |
| `tag-resource` | Manage table tags |
| CloudWatch `AWS/Cassandra` | Read/write throttling, latency, errors, capacity |
| `application-autoscaling` | Check/configure auto-scaling for provisioned tables |
| `cqlsh` with SigV4 | Execute CQL queries for data-level troubleshooting |

## Gotchas: Amazon Keyspaces

- Keyspaces is NOT Apache Cassandra. It is a serverless, CQL-compatible service with a different underlying engine. Many Cassandra-native features are unavailable. Do not assume Cassandra documentation applies directly.
- No ALLOW FILTERING by default. Queries must use partition key. Full table scans via ALLOW FILTERING are not supported in the same way as open-source Cassandra. Queries must be designed around partition keys.
- No lightweight transactions (LWT) in some versions. Conditional writes (IF NOT EXISTS, IF conditions) have limited support. Check current documentation for LWT availability in your region.
- Partition key design is critical — same principles as DynamoDB. Hot partitions cause throttling. Distribute writes evenly across partition keys. Single hot partition can throttle the entire table.
- TLS is required for all connections. Keyspaces does not accept unencrypted connections. Port 9142 with TLS. The Starfield digital certificate must be trusted by the client.
- SigV4 or service-specific credentials for authentication. IAM-based SigV4 authentication is recommended. Service-specific credentials (username/password) are an alternative generated via IAM console. No native Cassandra authentication.
- No UDFs, UDAs, or materialized views. User-defined functions, user-defined aggregates, and materialized views are not supported. Use application-side logic or secondary indexes instead.
- TTL maximum is 630,720,000 seconds (20 years). TTL values exceeding this are rejected. TTL is set per-row, not per-table. Expired data is deleted asynchronously and may be visible briefly after expiration.
- PITR provides continuous backup. Point-in-time recovery is always-on continuous backup with 35-day retention. Restores create a new table. No manual snapshots like Cassandra nodetool.
- No multi-region tables natively. Use application-level replication patterns. Keyspaces does not support Cassandra-style multi-datacenter replication. Each table exists in a single region.

## Anti-hallucination rules

1. Always cite specific table names, keyspace names, or API responses as evidence.
2. Keyspaces is NOT Cassandra. Never suggest Cassandra-specific tools (nodetool, cassandra-stress) or features (materialized views, UDFs).
3. TLS is mandatory. Never suggest connecting without TLS or on port 9042.
4. Partition key design drives performance. Never ignore partition key analysis for throttling issues.
5. CQL compatibility is partial. Never assume all CQL features work. Verify against Keyspaces documentation.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Table | A1-A3 | Creation failures, schema issues, capacity mode |
| B — Performance | B1-B3 | Read/write throttling, hot partitions, query performance |
| C — CQL | C1-C2 | CQL compatibility, unsupported features |
| D — Connectivity | D1-D2 | Connection failures, TLS/SigV4 auth |
| E — Capacity | E1-E2 | On-demand vs provisioned, auto-scaling |
| F — TTL | F1-F2 | TTL configuration, expiration behavior |
| G — Backup | G1-G2 | PITR, snapshot restore |
| H — Migration | H1-H2 | Cassandra migration, CQLReplicator |
| Z — Catch-All | Z1 | General troubleshooting |
