---
title: "B3 — SerDe Errors"
description: "Diagnose and resolve Athena SerDe deserialization errors"
status: active
severity: HIGH
triggers:
  - "SerDe"
  - "deserialization"
  - "HIVE_CURSOR_ERROR"
  - "JsonParseException"
  - "OpenCSVSerDe"
owner: devops-agent
objective: "Identify SerDe configuration errors and fix data deserialization"
context: "SerDe (Serializer/Deserializer) translates between S3 file bytes and Athena's row/column format. Wrong SerDe selection or configuration causes parse failures."
---

## Phase 1 — Triage

MUST:
- Check current SerDe: `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.SerdeInfo'`
- Check SerDe parameters: `aws glue get-table --database-name <db> --name <table> --query 'Table.StorageDescriptor.SerdeInfo.Parameters'`
- Sample actual data format: `aws s3 cp <table-location>/sample-file - | head -3`
- Verify column count matches data fields

SHOULD:
- Check if data has headers that should be skipped (skip.header.line.count)
- Verify delimiter, quote character, and escape character settings
- Check for encoding issues (UTF-8 vs other encodings)

MAY:
- Test with a minimal table definition on a single file
- Check if the SerDe library JAR is available (for custom SerDes)

## Phase 2 — Remediate

MUST:
- If wrong SerDe: update to correct one (OpenCSVSerDe for CSV, JsonSerDe for JSON, etc.)
- If wrong parameters: fix delimiter, quote char, or other SerDe-specific settings
- If header row issue: add 'skip.header.line.count'='1' to table properties
- If encoding issue: ensure data is UTF-8 encoded

SHOULD:
- Use org.apache.hadoop.hive.serde2.OpenCSVSerDe for quoted CSV
- Use org.openx.data.jsonserde.JsonSerDe for JSON with nested structures
- Use org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe for Parquet

MAY:
- Use ROW FORMAT DELIMITED for simple delimited files
- Consider Grok SerDe for log files with complex patterns

## Common Issues

- symptoms: "JsonParseException: Unexpected character at position 0"
  diagnosis: "JsonSerDe used on non-JSON data (CSV, TSV, or other format)."
  resolution: "Change SerDe to match actual format. Use OpenCSVSerDe for CSV."

- symptoms: "Number of columns in data does not match schema"
  diagnosis: "CSV has different number of fields than Glue table columns."
  resolution: "Update Glue table to match actual column count. Check for delimiter issues."

- symptoms: "HIVE_CURSOR_ERROR: Row is not a valid JSON object"
  diagnosis: "JSON data has one object per line but SerDe expects array format, or vice versa."
  resolution: "Use JsonSerDe for JSON Lines format. Ensure each line is a valid JSON object."

## Output Format

```yaml
root_cause: "SerDe error — <specific_cause>"
evidence:
  - type: serde_config
    content: "<current SerDe and parameters>"
  - type: data_sample
    content: "<actual data format>"
severity: HIGH
mitigation:
  immediate: "Update Glue table SerDe to match actual data format"
  long_term: "Standardize data format and validate SerDe config in ETL pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Update to correct SerDe | YELLOW | Modifies catalog metadata — may affect other queries |
| Fix delimiter/quote/escape settings | YELLOW | SerDe parameter change — affects data interpretation |
| Add skip.header.line.count property | GREEN | Table property addition — non-destructive |
| Ensure data is UTF-8 encoded | YELLOW | May require data re-encoding |
| Use ROW FORMAT DELIMITED for simple files | YELLOW | SerDe change — affects all queries on table |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- SerDe changes affect all queries reading from the table
- Data re-encoding may require ETL pipeline changes

## Data Sensitivity

- Data sampling to verify format may expose sensitive content
- SerDe configuration reveals data structure and format details
- Error messages may contain data snippets from failed deserialization

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER change SerDe on production tables without testing on a copy first
- NEVER assume data encoding without verifying with actual file samples

## Phase 3 — Rollback

- If SerDe was changed incorrectly: restore previous SerDe using `aws glue update-table`
- If SerDe parameters were modified: revert to original parameter values
- If table properties were added incorrectly: remove with `ALTER TABLE UNSET TBLPROPERTIES`
- Document all Glue table changes for audit trail

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
