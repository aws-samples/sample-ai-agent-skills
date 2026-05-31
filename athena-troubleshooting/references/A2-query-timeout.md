---
title: "A2 — Query Timeout"
description: "Diagnose and resolve Athena query timeout issues"
status: active
severity: HIGH
triggers:
  - "query timeout"
  - "query cancelled"
  - "exceeded time limit"
  - "30 minutes"
  - "slow query"
owner: devops-agent
objective: "Identify why the Athena query is timing out and optimize for completion within limits"
context: "Athena queries have a default 30-minute timeout configurable per workgroup. Timeouts typically indicate unoptimized queries scanning too much data, missing partitions, or inefficient joins."
---

## Phase 1 — Triage

MUST:
- Get query execution with timing: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.{Status:Status,Stats:Statistics}'`
- Check data scanned: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.Statistics.DataScannedInBytes'`
- Check workgroup timeout setting: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration'`
- Verify table partitioning: `aws glue get-table --database-name <db> --name <table> --query 'Table.PartitionKeys'`

SHOULD:
- Check partition count: `aws glue get-partitions --database-name <db> --table-name <table> --query 'Partitions | length(@)'`
- Verify data format (CSV vs Parquet/ORC): `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.SerdeInfo'`
- Check query runtime statistics: `aws athena get-query-runtime-statistics --query-execution-id <query-id>`
- Review the query for full table scans or missing WHERE clauses

MAY:
- Use EXPLAIN to analyze query plan: run `EXPLAIN <query>` in Athena
- Check S3 file sizes and count in the table location
- Compare with similar queries that completed successfully

## Phase 2 — Remediate

MUST:
- If full table scan on CSV: convert to Parquet/ORC using CTAS
- If no partition pruning: add partition filters to WHERE clause
- If too many small files: compact files using CTAS or Glue ETL
- If workgroup timeout too low: increase timeout in workgroup configuration

SHOULD:
- Add partitioning to unpartitioned tables (by date, region, etc.)
- Enable partition projection for time-series data
- Use columnar formats with compression (Parquet + Snappy)

MAY:
- Split complex queries into multiple simpler queries
- Use CTAS to materialize intermediate results
- Consider provisioned capacity for predictable large workloads

## Common Issues

- symptoms: "Query cancelled after 30 minutes with no results"
  diagnosis: "Query scanning too much data without partition pruning."
  resolution: "Add partition filters. Convert to Parquet. Use LIMIT for exploratory queries."

- symptoms: "Query runs for 25+ minutes on a small dataset"
  diagnosis: "Too many small files causing excessive S3 LIST operations."
  resolution: "Compact small files into larger ones (128MB-512MB) using CTAS or Glue ETL."

## Output Format

```yaml
root_cause: "Query timeout — <specific_cause>"
evidence:
  - type: query_statistics
    content: "<data scanned, runtime, timeout setting>"
  - type: table_definition
    content: "<partitioning, format, file count>"
severity: HIGH
mitigation:
  immediate: "Optimize query with partition filters and columnar format"
  long_term: "Implement partitioning strategy and data compaction pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Convert to Parquet/ORC using CTAS | YELLOW | Creates new table/data — verify before switching |
| Add partition filters to WHERE clause | GREEN | Query modification only |
| Compact files using CTAS or Glue ETL | YELLOW | Creates new data files — original preserved |
| Increase timeout in workgroup configuration | YELLOW | Modifies workgroup settings affecting all users |
| Add partitioning to unpartitioned tables | YELLOW | Structural change to table metadata |
| Enable partition projection | YELLOW | Modifies table properties |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- CTAS operations on large datasets may incur significant costs
- Workgroup timeout changes affect all queries in the workgroup

## Data Sensitivity

- Query optimization may require examining query text containing sensitive filter values
- CTAS operations create copies of data — ensure destination has equivalent encryption
- Workgroup configuration changes may affect data scan limits protecting cost controls
- Partition projection configuration may reveal data organization patterns

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER increase workgroup timeout without understanding cost implications
- NEVER remove data scan limits without approval from cost management
- NEVER suggest dropping partitions to fix timeout issues

## Phase 3 — Rollback

- If workgroup timeout was changed: restore original timeout value using `aws athena update-work-group`
- If CTAS created incorrect table: drop the new table and remove S3 data at the new location
- If partition projection was misconfigured: remove projection properties from table using `ALTER TABLE UNSET TBLPROPERTIES`
- If file compaction produced bad results: restore original table location reference
- Document all workgroup and table changes for audit trail

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
  - "NEVER suggest disabling query result encryption"
  - "NEVER suggest overly broad S3 access for workgroups"
  - "NEVER suggest disabling workgroup enforcement"
