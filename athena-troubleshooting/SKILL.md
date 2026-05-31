---
name: athena-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Athena problems
  by analyzing query failures, timeout issues, syntax errors, performance
  degradation, data format problems, partitioning, SerDe errors, workgroup
  configuration, federated queries, cost optimization, Glue Catalog
  integration, Lake Formation permissions, IAM access, encryption, and
  Athena for Apache Spark issues. Activate when: query failures, query
  timeouts, syntax errors, slow queries, data format mismatches, partition
  projection issues, SerDe deserialization errors, workgroup misconfiguration,
  query result location errors, federated query failures, data source
  connector issues, high data scan costs, query result reuse problems,
  Glue Data Catalog sync issues, Lake Formation permission denials, IAM
  permission errors, encryption failures, Spark session issues, notebook
  errors, or the user says something is wrong with Athena without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI with athena, glue, s3, lakeformation, and cloudtrail
  permissions. Optional: Athena JDBC/ODBC driver for connectivity testing,
  AWS Glue console access for catalog verification.
---

# Athena Diagnostics

## When to use

Any Athena investigation where the console alone is insufficient — query execution failures, performance issues, data format problems, workgroup configuration, federated queries, cost optimization, catalog integration, security permissions, or Spark notebook issues.

## Investigation workflow

### Step 1 — Collect and triage

```
# List workgroups and their configuration
aws athena list-work-groups --query 'WorkGroups[*].{Name:Name,State:State,Engine:EngineVersion.SelectedEngineVersion}'
version: "1.0.0"
last_updated: "2025-04-12"

# Get workgroup details
aws athena get-work-group --work-group <workgroup-name>

# List recent query executions
aws athena list-query-executions --work-group <workgroup-name> --max-items 10

# Get query execution details (status, data scanned, runtime)
aws athena get-query-execution --query-execution-id <query-id>

# Batch get multiple query executions
aws athena batch-get-query-execution --query-execution-ids <id1> <id2> <id3>

# Check Glue Data Catalog databases
aws glue get-databases --query 'DatabaseList[*].{Name:Name,Location:LocationUri}'
version: "1.0.0"
last_updated: "2025-04-12"

# Check specific table definition
aws glue get-table --database-name <db> --name <table>
```

Triage returns:
- Workgroup state, engine version, configuration
- Query execution status, error messages, data scanned
- Runtime duration and query type
- Glue catalog database and table definitions

If the query status is FAILED, the StateChangeReason IS the root cause. Don't chase infrastructure symptoms.

### Step 2 — Domain deep dive (only if needed)

```
# Get query execution statistics
aws athena get-query-runtime-statistics --query-execution-id <query-id>

# Check S3 output location accessibility
aws s3 ls <output-location>

# Check table partitions
aws glue get-partitions --database-name <db> --table-name <table> --max-items 10

# Check Lake Formation permissions
aws lakeformation list-permissions --principal DataLakePrincipalIdentifier=<arn>

# Check data source connectors (federated queries)
aws athena list-data-catalogs

# CloudTrail for Athena API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=athena.amazonaws.com --max-results 20
```

Read `references/athena-guardrails.md` before concluding on any Athena issue.

### Step 3 — Detailed investigation (low-confidence cases only)

```
# Check named queries for reference
aws athena list-named-queries --work-group <workgroup-name>

# Check prepared statements
aws athena list-prepared-statements --work-group <workgroup-name>

# Verify S3 bucket policy for query results
aws s3api get-bucket-policy --bucket <results-bucket>

# Check KMS key access for encrypted results
aws kms describe-key --key-id <key-id>
aws kms list-grants --key-id <key-id>

# IAM policy simulation for Athena actions
aws iam simulate-principal-policy --policy-source-arn <role-arn> \
  --action-names athena:StartQueryExecution athena:GetQueryResults s3:GetObject glue:GetTable

# Check Glue crawler status (if partitions are stale)
aws glue get-crawler --name <crawler-name>
```

## Tool quick reference

| Tool / Command | When to use |
|----------------|-------------|
| `aws athena get-query-execution` | Query status, error message, data scanned, runtime |
| `aws athena get-work-group` | Workgroup config, output location, engine version |
| `aws athena list-data-catalogs` | List federated data sources and connectors |
| `aws glue get-table` | Table schema, SerDe, input format, location |
| `aws glue get-partitions` | Partition keys, values, storage descriptors |
| `aws lakeformation list-permissions` | Lake Formation grants for a principal |
| `aws s3 ls` | Verify S3 data/output location accessibility |
| `aws cloudtrail lookup-events` | Athena API call history and errors |
| `aws iam simulate-principal-policy` | Verify IAM permissions for Athena operations |
| `aws kms describe-key` | KMS key status and configuration |

## Gotchas: Athena

These are the mistakes commonly made during Athena troubleshooting.

- Athena charges per TB of data scanned — not per query or per time. Use partitioning, columnar formats (Parquet/ORC), and compression to reduce costs. A full table scan on CSV data is the most expensive operation.
- Default query timeout is 30 minutes. This is configurable per workgroup (up to the service maximum). Long-running queries that exceed the timeout are cancelled automatically.
- Workgroups control cost, access, and configuration. Each workgroup has its own query result location, encryption settings, data scan limits, and engine version. Users inherit workgroup settings.
- Glue Data Catalog is the metastore for Athena. Tables, databases, and partitions are defined in Glue. If the Glue table definition is wrong (wrong SerDe, wrong location, wrong schema), Athena queries fail.
- CTAS (CREATE TABLE AS SELECT) and INSERT INTO are used for ETL in Athena. CTAS creates a new table from query results. INSERT INTO appends to an existing table. Both write to S3 in the specified format.
- Federated queries use Lambda-based connectors to query external data sources (RDS, DynamoDB, Redis, etc.). The Lambda function must have network access to the source and IAM permissions. Federated queries are slower than native S3 queries.
- The query result location must be writable by the executing principal. If the S3 bucket policy or IAM policy doesn't allow PutObject, the query fails even if the SELECT itself succeeds.
- Athena engine v3 is based on Trino (formerly PrestoSQL). Engine v2 is based on Presto. There are syntax and function differences between versions. Check the workgroup engine version when debugging syntax errors.
- Partition projection eliminates the need for MSCK REPAIR TABLE or Glue crawlers by computing partition values on the fly. But it requires correct configuration in table properties — wrong projection config causes empty results, not errors.
- SerDe (Serializer/Deserializer) must match the data format. Using OpenCSVSerDe on JSON data or JsonSerDe on CSV data produces NULL values or parse errors. Always verify the SerDe matches the actual file format.
- S3 path patterns matter. Athena reads ALL files in the table's S3 location. Stray files (like _SUCCESS markers, hidden files, or wrong-format files) cause query failures or incorrect results.
- Athena does not support UPDATE or DELETE on regular tables. For mutable data, use Iceberg table format which supports ACID transactions in Athena engine v3.

## Anti-hallucination rules

1. Always cite specific query execution IDs, error messages, Glue table definitions, or S3 paths as evidence.
2. Never suggest SSH-ing into Athena infrastructure — it is fully serverless with no instances to access.
3. Never recommend changing Athena's pricing model — it is always per-TB-scanned (or per-query for provisioned capacity).
4. Never claim Athena supports UPDATE/DELETE on regular tables — only Iceberg tables support DML.
5. Never suggest Athena has persistent compute resources — each query runs on ephemeral compute.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

Runbooks are organized by failure domain. Use the appropriate runbook based on the symptom category.

| Category | IDs | Covers |
|----------|-----|--------|
| A — Query | A1–A4 | Query failures, timeout, syntax errors, performance |
| B — Data | B1–B3 | Data format issues, partitioning, SerDe errors |
| C — Workgroup | C1–C2 | Workgroup configuration, query result location |
| D — Federation | D1–D2 | Federated query, data source connectors |
| E — Cost | E1–E2 | Data scanned optimization, query result reuse |
| F — Integration | F1–F2 | Glue Catalog, Lake Formation |
| G — Security | G1–G2 | IAM permissions, encryption |
| H — Spark | H1–H2 | Athena Spark, notebook issues |
| Z — Catch-All | Z1 | General Athena troubleshooting |
