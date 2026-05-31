---
title: "E1 — Data Scanned Optimization"
description: "Optimize Athena costs by reducing data scanned per query"
status: active
severity: MEDIUM
triggers:
  - "cost optimization"
  - "data scanned"
  - "expensive queries"
  - "reduce cost"
  - "TB scanned"
owner: devops-agent
objective: "Identify high-cost queries and implement data scan reduction strategies"
context: "Athena charges $5/TB scanned. Optimization through partitioning, columnar formats, compression, and query design can reduce costs by 90%+."
---

## Phase 1 — Triage

MUST:
- Get recent query data scanned: `aws athena list-query-executions --work-group <workgroup-name> --max-items 20` then batch get statistics
- Check table format: `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.SerdeInfo.SerializationLibrary'`
- Check partitioning: `aws glue get-table --database-name <db> --name <table> --query 'Table.PartitionKeys'`
- Check total data size: `aws s3 ls <table-location> --recursive --summarize | tail -2`

SHOULD:
- Identify top queries by data scanned from query history
- Check if columnar format is used (Parquet/ORC vs CSV/JSON)
- Verify partition pruning effectiveness in common queries

MAY:
- Analyze workgroup-level cost trends via CloudWatch
- Check if query result reuse is enabled

## Phase 2 — Remediate

MUST:
- If CSV/JSON: convert to Parquet using CTAS: `CREATE TABLE new_table WITH (format='PARQUET', external_location='s3://...') AS SELECT * FROM old_table`
- If unpartitioned: add partitioning by common filter columns
- If no compression: enable Snappy compression for Parquet
- If SELECT *: rewrite queries to select only needed columns

SHOULD:
- Enable workgroup data scan limits as cost guardrails
- Use partition projection for time-series data
- Implement bucketing for frequently filtered columns

MAY:
- Set up cost anomaly detection with AWS Cost Explorer
- Use provisioned capacity for predictable high-volume workloads

## Common Issues

- symptoms: "Monthly Athena bill increased 5x"
  diagnosis: "New queries scanning full unpartitioned CSV tables."
  resolution: "Convert to Parquet with partitioning. Add data scan limits to workgroup."

- symptoms: "Single query scans 10TB of data"
  diagnosis: "Query missing partition filter on a large partitioned table."
  resolution: "Add WHERE clause on partition columns to enable partition pruning."

## Output Format

```yaml
root_cause: "High data scan cost — <specific_cause>"
evidence:
  - type: query_statistics
    content: "<data scanned per query>"
  - type: table_format
    content: "<current format and partitioning>"
severity: MEDIUM
mitigation:
  immediate: "Convert to columnar format and add partition filters"
  long_term: "Implement data pipeline with Parquet, partitioning, and compression"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Convert to Parquet using CTAS | YELLOW | Creates new table/data — verify before switching |
| Add partitioning by common filter columns | YELLOW | Structural change to table metadata |
| Enable Snappy compression | GREEN | Applied during CTAS — non-destructive |
| Rewrite queries to select only needed columns | GREEN | Query modification only |
| Enable workgroup data scan limits | GREEN | Adds cost guardrail — protective measure |
| Use partition projection | YELLOW | Modifies table properties |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Cost optimization changes may affect query performance SLAs
- Data format conversion on large datasets may incur significant one-time costs

## Data Sensitivity

- Cost analysis may reveal data volumes and query patterns
- CTAS operations create data copies — ensure equivalent encryption at destination
- Query history analysis may reveal sensitive query patterns and data access
- Data scan statistics may reveal dataset sizes and growth patterns

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER remove data scan limits without cost management approval
- NEVER delete original data before verifying converted data integrity and completeness

## Phase 3 — Rollback

- If CTAS created incorrect table: drop the new table and remove S3 data at the new location
- If partitioning was added incorrectly: restore original table definition
- If workgroup data scan limits were set too restrictively: adjust limits upward
- If partition projection was misconfigured: remove projection properties
- Document all optimization changes for audit trail

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
