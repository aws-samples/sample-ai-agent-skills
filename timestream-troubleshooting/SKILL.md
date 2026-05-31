---
name: timestream-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Timestream problems
  by analyzing table creation failures, retention configuration, write failures,
  rejected records, batch write errors, query failures, query performance,
  scheduled queries, memory store issues, magnetic store configuration,
  Grafana integration, IoT Core integration, IAM permissions, VPC endpoints,
  and following structured runbooks. Activate when: table creation failures,
  retention policy misconfiguration, write ingestion errors, rejected records,
  batch write failures, query syntax errors, slow queries, scheduled query
  failures, memory store retention issues, magnetic store write problems,
  Grafana plugin issues, IoT Core rule action failures, IAM access denied,
  VPC endpoint configuration errors, or the user says something is wrong
  with Timestream without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Timestream Write, Timestream Query,
  CloudWatch, IAM, CloudTrail, and optionally KMS permissions. Some operations
  require timestream:DescribeEndpoints for endpoint discovery.
---

# Amazon Timestream Diagnostics

## When to use

Any Amazon Timestream investigation where the console alone is insufficient — table creation, retention policies, write ingestion failures, rejected records, query errors, scheduled queries, memory/magnetic store configuration, IAM permissions, or integration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws timestream-write describe-database --database-name <database>
aws timestream-write describe-table --database-name <database> --table-name <table>
aws timestream-write list-databases
aws timestream-write list-tables --database-name <database>
```

### Step 2 — Domain deep dive

```
aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name SystemErrors --dimensions Name=DatabaseName,Value=<database> Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Timestream --metric-name UserErrors --dimensions Name=DatabaseName,Value=<database> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws timestream-query query --query-string "SELECT count(*) FROM \"<database>\".\"<table>\" WHERE time > ago(1h)"
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=timestream.amazonaws.com --max-results 20
aws timestream-write describe-endpoints
aws kms describe-key --key-id <key-id>
```

Read `references/timestream-guardrails.md` before concluding on any Timestream issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-database` | Check database status and configuration |
| `describe-table` | Check table retention, magnetic store, status |
| `describe-endpoints` | Get Timestream service endpoints |
| `timestream-query query` | Execute SQL queries |
| `list-databases` / `list-tables` | Enumerate resources |
| CloudWatch metrics | Monitor SystemErrors, UserErrors, write latency |
| CloudTrail | Audit API calls and configuration changes |

## Gotchas: Amazon Timestream

- Memory store retention is 1 hour to 8760 hours (1 year). Magnetic store retention is 1 day to 73000 days (200 years). These are separate settings. Data transitions from memory to magnetic automatically when it ages past the memory store retention window.
- Late-arriving data is rejected if the timestamp is older than the memory store retention window, UNLESS magnetic store writes are enabled on the table. Enable magnetic store writes to accept late-arriving data directly into magnetic store.
- Timestream does not support UPDATE or DELETE operations. Data is append-only and immutable. The only way to remove data is to let it expire via retention policies. Plan retention carefully before writing data.
- Multi-measure records allow up to 256 measures per record and are more efficient than single-measure records. Mixing single and multi-measure in the same table is not recommended.
- Scheduled queries materialize results into a destination table. They require IAM roles, SNS topics for notifications, and S3 for error reports. Failures are silent without SNS configuration.
- Dimensions are indexed and used for query filtering. Choose dimensions carefully — they cannot be changed after data is written. Too many unique dimension combinations cause high cardinality issues.
- Timestream for InfluxDB is a completely different service from Timestream for LiveAnalytics. They have different APIs, pricing, and capabilities. Do not confuse the two.
- Endpoint discovery is required. Clients must call DescribeEndpoints to get the correct write/query endpoints. SDK handles this automatically but custom HTTP clients must implement it.

## Anti-hallucination rules

1. Always cite specific database/table names, error messages, or API responses as evidence.
2. Memory store retention is in HOURS, magnetic store retention is in DAYS. Never confuse units.
3. Timestream data is immutable — no UPDATE or DELETE. Never suggest modifying existing records.
4. Endpoint discovery is required. Never hardcode Timestream endpoints.
5. Magnetic store writes must be explicitly enabled. Never assume they are on by default.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Table | A1–A2 | Creation failures, retention configuration |
| B — Write | B1–B3 | Write failures, rejected records, batch write errors |
| C — Query | C1–C3 | Query failures, performance, scheduled queries |
| D — Memory/Magnetic | D1–D2 | Memory store, magnetic store |
| E — Integration | E1–E2 | Grafana, IoT Core |
| F — Security | F1–F2 | IAM permissions, VPC endpoints |
| Z — Catch-All | Z1 | General troubleshooting |
