---
title: "B1 — Data Format Issues"
description: "Diagnose and resolve Athena data format mismatch problems"
status: active
severity: HIGH
triggers:
  - "data format"
  - "HIVE_BAD_DATA"
  - "wrong format"
  - "NULL values"
  - "empty results"
owner: devops-agent
objective: "Identify data format mismatches between S3 files and Glue table definitions"
context: "Athena relies on the Glue table SerDe to interpret S3 data. Format mismatches cause NULL values, parse errors, or HIVE_BAD_DATA exceptions."
---

## Phase 1 — Triage

MUST:
- Check Glue table SerDe: `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.{SerDe:SerdeInfo,InputFormat:InputFormat,OutputFormat:OutputFormat}'`
- Sample actual S3 data to verify format: `aws s3 cp <table-location>/sample-file - | head -5`
- Check for mixed file formats in the location: `aws s3 ls <table-location> --recursive | head -20`
- Verify column definitions match data: `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.Columns'`

SHOULD:
- Check for stray files (_SUCCESS, .crc, hidden files): `aws s3 ls <table-location> --recursive | grep -E '_SUCCESS|\.crc|\._'`
- Verify compression codec matches table properties
- Check if data was recently changed by an ETL job

MAY:
- Use S3 Select to preview file contents without full download
- Check Glue crawler configuration if schema was auto-detected

## Phase 2 — Remediate

MUST:
- If SerDe mismatch: update Glue table to use correct SerDe for the actual data format
- If mixed formats: separate files by format or convert all to a single format
- If stray files: remove non-data files from the table location
- If schema mismatch: update column definitions to match actual data

SHOULD:
- Convert to Parquet/ORC for better performance and type safety
- Use Glue ETL to standardize data format across all files
- Set up data validation in the ingestion pipeline

MAY:
- Use IGNORE_MALFORMED_DATA table property as temporary workaround
- Create a new table with correct definition and CTAS to migrate data

## Common Issues

- symptoms: "All columns return NULL for CSV data"
  diagnosis: "Wrong delimiter in SerDe properties or wrong SerDe for the file format."
  resolution: "Check actual delimiter in data. Update SerDe properties: 'separatorChar'=',' or use correct SerDe."

- symptoms: "HIVE_BAD_DATA: Error parsing field value for field X"
  diagnosis: "Data type in file does not match column type in Glue table."
  resolution: "Update column type in Glue table or fix data in the pipeline."

## Output Format

```yaml
root_cause: "Data format issue — <specific_mismatch>"
evidence:
  - type: glue_table
    content: "<SerDe and format configuration>"
  - type: s3_data
    content: "<actual file format and sample>"
severity: HIGH
mitigation:
  immediate: "Fix Glue table definition to match actual data format"
  long_term: "Standardize on columnar format with schema validation in ETL pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update Glue table to correct SerDe | YELLOW | Modifies catalog metadata — may affect other queries |
| Separate files by format | YELLOW | Data reorganization — verify integrity |
| Remove non-data files from table location | YELLOW | Deletes files from S3 — ensure they are truly non-data |
| Update column definitions | YELLOW | Schema change — may affect downstream queries |
| Convert to Parquet/ORC | YELLOW | Creates new data — original preserved |
| Use IGNORE_MALFORMED_DATA property | GREEN | Table property — temporary workaround |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Glue table changes affect shared catalog used by multiple applications
- S3 file removal is irreversible without versioning

## Data Sensitivity

- Data sampling during diagnosis may expose sensitive content — use LIMIT and column selection
- Glue catalog changes may affect data access patterns for sensitive tables
- S3 file listing may reveal data organization containing sensitive information

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER delete S3 data files without confirming they are non-data artifacts
- NEVER modify Glue table schema without understanding all dependent queries and applications

## Phase 3 — Rollback

- If Glue table SerDe was changed: restore previous SerDe configuration using `aws glue update-table`
- If column definitions were modified: revert to original schema
- If non-data files were removed incorrectly: restore from S3 versioning if enabled
- If data format conversion failed: restore original table definition and location
- Document all Glue catalog and S3 changes for audit trail

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
