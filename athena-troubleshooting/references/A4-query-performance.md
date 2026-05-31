---
title: "A4 — Query Performance"
description: "Diagnose and resolve Athena query performance issues"
status: active
severity: MEDIUM
triggers:
  - "slow query"
  - "query performance"
  - "takes too long"
  - "data scanned"
  - "optimization"
owner: devops-agent
objective: "Identify performance bottlenecks and optimize Athena query execution"
context: "Athena performance depends on data format, partitioning, file sizes, query structure, and data volume. Optimization reduces both execution time and cost."
---

## Phase 1 — Triage

MUST:
- Get query statistics: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.Statistics'`
- Get runtime statistics: `aws athena get-query-runtime-statistics --query-execution-id <query-id>`
- Check table format and partitioning: `aws glue get-table --database-name <db> --name <table> --query 'Table.{SerDe:StorageDescriptor.SerdeInfo.SerializationLibrary,Partitions:PartitionKeys,Location:StorageDescriptor.Location}'`
- Check data scanned vs total table size

SHOULD:
- Run EXPLAIN on the query to see the execution plan
- Check file sizes in S3: `aws s3 ls <table-location> --recursive --summarize`
- Verify partition pruning is effective by checking data scanned with and without filters
- Check for data skew in partitions

MAY:
- Compare performance across engine versions (v2 vs v3)
- Check if query result reuse could help for repeated queries
- Analyze join strategies for multi-table queries

## Phase 2 — Remediate

MUST:
- If CSV format: convert to Parquet or ORC using CTAS
- If no partitioning: add partitioning by high-cardinality filter columns
- If many small files (<128MB): compact using CTAS or Glue ETL
- If few huge files (>1GB): split into optimal sizes (128MB-512MB)

SHOULD:
- Enable partition projection for time-series partitions
- Use columnar predicates (WHERE on partition columns first)
- Add compression (Snappy for Parquet, Zlib for ORC)

MAY:
- Use bucketing for frequently joined columns
- Enable query result reuse in workgroup settings
- Consider provisioned capacity for predictable workloads

## Common Issues

- symptoms: "Query scans 500GB but only needs 1 day of data"
  diagnosis: "Table not partitioned or partition filters missing in WHERE clause."
  resolution: "Add partitioning by date. Include partition column in WHERE clause."

- symptoms: "Query on Parquet table still slow despite partitioning"
  diagnosis: "Too many small Parquet files causing overhead."
  resolution: "Compact files to 128MB-512MB using CTAS with bucketing."

## Output Format

```yaml
root_cause: "Query performance — <specific_bottleneck>"
evidence:
  - type: query_statistics
    content: "<data scanned, runtime, planning time>"
  - type: table_format
    content: "<format, partitioning, file sizes>"
severity: MEDIUM
mitigation:
  immediate: "Apply the identified optimization (format conversion, partitioning, compaction)"
  long_term: "Implement data pipeline with optimal format, partitioning, and file sizing"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Convert CSV to Parquet/ORC using CTAS | YELLOW | Creates new table/data — verify before switching |
| Add partitioning by high-cardinality columns | YELLOW | Structural change to table metadata |
| Compact small files using CTAS or Glue ETL | YELLOW | Creates new data files — original preserved |
| Split huge files into optimal sizes | YELLOW | Data reorganization — verify integrity |
| Enable partition projection | YELLOW | Modifies table properties |
| Enable query result reuse | GREEN | Workgroup setting — no data modification |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Data format conversion on large datasets may incur significant costs
- Performance changes may affect SLA-bound queries

## Data Sensitivity

- Performance diagnostics may require examining query text with sensitive filters
- CTAS operations create data copies — ensure equivalent encryption at destination
- Query statistics may reveal data volumes and access patterns
- File size analysis may require listing S3 objects in sensitive data locations

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER remove existing partitioning without understanding downstream query impact
- NEVER suggest deleting original data before verifying converted data integrity

## Phase 3 — Rollback

- If CTAS created incorrect table: drop the new table and remove S3 data at the new location
- If partitioning was changed: restore original table definition from Glue version history
- If file compaction produced bad results: restore original table location reference
- If partition projection was misconfigured: remove projection properties using `ALTER TABLE UNSET TBLPROPERTIES`
- Document all changes for audit trail

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
