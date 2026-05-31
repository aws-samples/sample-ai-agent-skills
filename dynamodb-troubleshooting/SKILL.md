---
name: dynamodb-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon DynamoDB
  problems by analyzing table configuration, capacity, indexes,
  and following structured runbooks. Activate when: read/write
  throttling, hot partition issues, GSI backpressure, query
  performance problems, capacity planning issues, DynamoDB Streams
  failures, TTL not expiring items, backup/restore issues, global
  table replication lag, transaction failures, DAX cache issues,
  table stuck in CREATING/UPDATING, or the user says something is
  wrong with DynamoDB without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with DynamoDB, CloudWatch,
  Application Auto Scaling, and optionally DAX permissions.
---

# DynamoDB Diagnostics

## When to use

Any DynamoDB investigation where CloudWatch metrics alone are insufficient — throttling root cause analysis, partition key design evaluation, GSI capacity issues, Streams processing failures, transaction conflicts, DAX caching problems, or global table replication issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws dynamodb describe-table --table-name <table>
aws dynamodb describe-time-to-live --table-name <table>
aws dynamodb describe-continuous-backups --table-name <table>
aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ThrottledRequests ...
aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedReadCapacityUnits ...
aws application-autoscaling describe-scaling-policies --service-namespace dynamodb --resource-id table/<table>
```

### Step 2 — Domain deep dive

```
aws dynamodb describe-table --table-name <table>  # → GSI status, partition info
aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> Name=GlobalSecondaryIndexName,Value=<gsi>
aws dynamodb describe-global-table --global-table-name <table>
aws dynamodb describe-export --export-arn <arn>
aws dynamodb list-streams --table-name <table>
```

Read `references/dynamodb-guardrails.md` before concluding on any DynamoDB issue.

## Gotchas: DynamoDB

- Provisioned capacity throttling occurs at the PARTITION level, not table level. A table with 1000 WCU spread across 10 partitions gives each partition ~100 WCU. A hot partition gets throttled even if table-level capacity is available.
- On-demand mode has initial burst capacity and scales based on previous peak. A sudden spike to 2x the previous peak may cause throttling until DynamoDB adapts.
- GSI backpressure: if a GSI is throttled, it throttles the BASE TABLE writes. The GSI is the bottleneck, not the table.
- GSI capacity is INDEPENDENT of the base table. A GSI needs its own provisioned capacity or on-demand scaling.
- Adaptive capacity automatically redistributes unused capacity to hot partitions, but it has limits and latency.
- Partition key design is the #1 cause of performance issues. High-cardinality, well-distributed keys are essential.
- Query returns up to 1 MB per call. Use pagination (LastEvaluatedKey) for larger result sets.
- Scan reads EVERY item in the table. It consumes capacity proportional to table size, not result size.
- Transactions are all-or-nothing across up to 100 items / 4 MB. They consume 2x the capacity of non-transactional operations.
- DynamoDB Streams records are available for 24 hours. If your consumer falls behind, records are lost.
- TTL deletion is best-effort and may take up to 48 hours after expiry. Don't rely on TTL for exact-time deletion.
- Global tables use last-writer-wins for conflict resolution. Concurrent writes to the same item in different regions may lose data.
- DAX is eventually consistent by default. Strongly consistent reads bypass DAX and go directly to DynamoDB.
- Item size limit is 400 KB. This includes attribute names and values.

### Capacity modes

| Mode | Behavior | Best For |
|------|----------|----------|
| Provisioned | Fixed RCU/WCU, auto-scaling optional | Predictable workloads |
| On-Demand | Pay-per-request, auto-scales | Unpredictable/spiky workloads |

### Consistency models

| Type | Behavior | Cost |
|------|----------|------|
| Eventually consistent | May return stale data (ms) | 1 RCU per 8 KB |
| Strongly consistent | Latest data guaranteed | 1 RCU per 4 KB |
| Transactional | ACID across items | 2 RCU per 4 KB |

## Anti-hallucination rules

1. Always cite specific CloudWatch metrics, table configuration, or API responses as evidence.
2. Throttling is per-partition, not per-table. Never diagnose throttling without considering partition key distribution.
3. GSI throttling causes base table write throttling. Never ignore GSI capacity when diagnosing write throttling.
4. TTL deletion can take up to 48 hours. Never claim TTL deletes items immediately at expiry time.
5. Scan consumes capacity proportional to table size, not result size. Never claim filtered scans are efficient.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 32 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Throttling | A1-A4 | Read throttling, write throttling, GSI backpressure, on-demand throttling |
| B — Partition & Key Design | B1-B3 | Hot partitions, key design, adaptive capacity |
| C — Query & Scan | C1-C3 | Query performance, scan optimization, pagination |
| D — Capacity | D1-D3 | Auto-scaling issues, provisioned vs on-demand, burst capacity |
| E — Indexes | E1-E3 | GSI creation/update, LSI limitations, projection issues |
| F — Streams | F1-F3 | Stream processing failures, Lambda triggers, consumer lag |
| G — TTL & Lifecycle | G1-G2 | TTL not expiring, TTL filter expressions |
| H — Global Tables | H1-H3 | Replication lag, conflict resolution, region failover |
| I — Transactions | I1-I2 | Transaction failures, idempotency |
| J — DAX | J1-J2 | Cache issues, consistency |
| Z — Catch-All | Z1 | General troubleshooting |
