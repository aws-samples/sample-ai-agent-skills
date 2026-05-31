---
title: "C3 — Spectrum External Table Issues"
description: "Diagnose and resolve Redshift Spectrum query failures"
status: active
severity: MEDIUM
triggers:
  - "Spectrum"
  - "external table"
  - "external schema"
  - "S3 query"
  - "Glue catalog"
  - "svl_s3query"
owner: devops-agent
objective: "Identify and fix Spectrum external table query failures"
context: "Redshift Spectrum queries data in S3 via external tables defined in an external schema (backed by Glue Data Catalog or Hive metastore). Spectrum uses compute nodes to scan S3 data. Performance depends on file format, file size, partitioning, and predicate pushdown."
---

## Phase 1 — Triage

MUST:
- Check Spectrum query errors: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, segment, elapsed, s3_scanned_rows, s3_scanned_bytes, files, avg_request_parallelism FROM svl_s3query_summary ORDER BY starttime DESC LIMIT 10"`
- Check external schema configuration: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM svv_external_schemas"`
- Verify IAM role for Spectrum: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].IamRoles'`
- Check Glue catalog table definition: `aws glue get-table --database-name <glue-db> --name <table-name>`

SHOULD:
- Check S3 file accessibility from the IAM role
- Verify partition registration: `SELECT * FROM svv_external_partitions WHERE tablename = '<table>'`
- Check for schema mismatch between Glue catalog and actual S3 data
- Check Spectrum scan metrics for performance issues

MAY:
- Check S3 file format and compression
- Verify predicate pushdown is working (check s3_scanned_bytes vs returned rows)
- Check for S3 request throttling (503 SlowDown)

## Phase 2 — Remediate

MUST:
- If permission denied: add s3:GetObject, s3:ListBucket, and glue:GetTable permissions to IAM role
- If schema mismatch: update Glue table definition or recreate external table
- If partition not found: run `ALTER TABLE <external_table> ADD PARTITION ...` or re-crawl with Glue crawler
- If file format error: verify SerDe matches actual file format

SHOULD:
- Use columnar formats (Parquet, ORC) for 10x better performance over CSV
- Ensure files are 100MB-1GB for optimal parallelism
- Add partitions to enable partition pruning in queries

MAY:
- Enable Spectrum result caching
- Use CREATE EXTERNAL TABLE AS to materialize Spectrum results
- Consider loading frequently queried data into Redshift native tables

## Common Issues

- symptoms: "Spectrum query returns 'S3 access denied'"
  diagnosis: "IAM role lacks S3 read permissions for the external table's S3 location."
  resolution: "Add s3:GetObject and s3:ListBucket to the IAM role for the S3 bucket."

- symptoms: "Spectrum query returns 0 rows but S3 has data"
  diagnosis: "Partitions not registered or partition filter excludes all data."
  resolution: "Check svv_external_partitions. Add missing partitions or run Glue crawler."

- symptoms: "Spectrum query is very slow scanning CSV files"
  diagnosis: "CSV requires full file scan. No predicate pushdown for CSV."
  resolution: "Convert to Parquet or ORC format. Add partitions for pruning."

## Output Format

```yaml
root_cause: "Spectrum issue — <specific_cause>"
evidence:
  - type: system_table
    content: "<svl_s3query_summary or svv_external_schemas output>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions, schema, or partition configuration"
  long_term: "Use columnar formats, implement partitioning, monitor Spectrum scan metrics"
```


## Safety Ratings
```
safety_ratings:
  - "Query svl_s3query_summary: GREEN — read-only system table query"
  - "Check external schema configuration: GREEN — read-only inspection"
  - "Check Glue catalog table: GREEN — read-only API call"
  - "Fix IAM role permissions: YELLOW — changes access scope for S3 and Glue"
  - "Add partitions to external table: GREEN — metadata-only change"
  - "Update Glue table definition: YELLOW — changes schema for all consumers of the Glue table"
  - "Recreate external table: YELLOW — drops and recreates table definition"
```

## Escalation Conditions
- Cluster serves production analytics
- Spectrum query failures blocking analytics pipelines
- Fix requires Glue catalog changes affecting other services
- IAM role changes needed for cross-account S3 access
- File format conversion needed for performance improvement

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "S3 data paths: external data source locations"
    - "Glue catalog metadata: table schemas and partition information"
    - "Query results: business analytics data from external sources"
    - "IAM role ARNs: permission identifiers"
  handling: "Do not expose S3 data paths or Glue catalog details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER modify Glue catalog tables without understanding impact on other services using the same catalog
- NEVER grant overly broad S3 permissions — scope to specific buckets and prefixes
- NEVER delete external table partitions without confirming the data is no longer needed for queries

## Phase 3 — Rollback
- If external table was recreated: recreate with original definition if the new one is incorrect
- If Glue table definition was updated: revert to previous table definition in Glue catalog
- If partitions were added: remove incorrect partitions via ALTER TABLE DROP PARTITION
- If IAM role permissions were changed: revert to previous IAM policy version

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
