---
title: "A1 — Query Failures"
description: "Diagnose and resolve Athena query execution failures"
status: active
severity: HIGH
triggers:
  - "query failed"
  - "FAILED status"
  - "query execution error"
  - "HIVE_"
  - "GENERIC_INTERNAL_ERROR"
owner: devops-agent
objective: "Identify why the Athena query failed and resolve the issue"
context: "Athena query failures produce specific error codes in the StateChangeReason field. Common causes include data format mismatches, permission errors, S3 access issues, and resource limits."
---

## Phase 1 — Triage

MUST:
- Get query execution details: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.{Status:Status,Query:Query,Stats:Statistics}'`
- Check the StateChangeReason for error message: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.Status.StateChangeReason'`
- Verify the workgroup configuration: `aws athena get-work-group --work-group <workgroup-name>`
- Check the Glue table definition: `aws glue get-table --database-name <db> --name <table> --query 'Table.{SerDe:StorageDescriptor.SerdeInfo,Location:StorageDescriptor.Location,Columns:StorageDescriptor.Columns}'`

SHOULD:
- Check S3 data location accessibility: `aws s3 ls <table-s3-location> --page-size 10`
- Verify IAM permissions for the executing role: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject s3:ListBucket glue:GetTable athena:StartQueryExecution`
- Check CloudTrail for related errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StartQueryExecution --max-results 5`
- Check if the query result location is writable: `aws s3 ls <result-location>`

MAY:
- Check recent successful queries in the same workgroup for comparison
- Review query runtime statistics: `aws athena get-query-runtime-statistics --query-execution-id <query-id>`
- Check AWS Health Dashboard for Athena service events

## Phase 2 — Remediate

MUST:
- If HIVE_CANNOT_OPEN_SPLIT: verify S3 file format matches Glue table SerDe definition
- If ACCESS_DENIED: fix IAM permissions for S3 data access and Glue catalog access
- If GENERIC_INTERNAL_ERROR: retry the query; if persistent, check for data corruption
- If result location error: ensure s3:PutObject permission on the result bucket

SHOULD:
- Fix Glue table definition if schema mismatch detected
- Update partition metadata if partition-related errors
- Use MSCK REPAIR TABLE or Glue crawler to sync partitions

MAY:
- Convert data to columnar format to avoid format-related failures
- Add error handling in application code for transient failures

## Common Issues

- symptoms: "HIVE_CANNOT_OPEN_SPLIT: Error opening Hive split"
  diagnosis: "Data file format does not match the SerDe in the Glue table definition."
  resolution: "Verify SerDe matches actual file format. Check for corrupted files in the S3 location."

- symptoms: "ACCESS_DENIED: User does not have access to the S3 path"
  diagnosis: "IAM policy or S3 bucket policy denies access to the data location."
  resolution: "Add s3:GetObject and s3:ListBucket permissions for the data S3 path."

- symptoms: "GENERIC_INTERNAL_ERROR: Null"
  diagnosis: "Transient internal error or data type mismatch causing null pointer."
  resolution: "Retry the query. If persistent, check for data type mismatches between schema and data."

- symptoms: "COLUMN_NOT_FOUND: Column 'x' cannot be resolved"
  diagnosis: "Query references a column not in the Glue table schema."
  resolution: "Check column names in Glue table definition. Column names are case-insensitive in Athena."

## Output Format

```yaml
root_cause: "Query failure — <specific_error_code_and_cause>"
evidence:
  - type: query_execution
    content: "<StateChangeReason error message>"
  - type: glue_table
    content: "<table definition details>"
severity: HIGH
mitigation:
  immediate: "Fix the identified data format, permission, or schema issue and re-run the query"
  long_term: "Implement data validation in ETL pipeline and pre-flight permission checks"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify S3 file format matches Glue table SerDe | GREEN | Read-only diagnostic check |
| Fix IAM permissions for S3/Glue access | YELLOW | Modifies IAM policies — review scope before applying |
| Retry query after fixing issue | GREEN | Re-executes existing query |
| Fix Glue table definition if schema mismatch | YELLOW | Modifies catalog metadata — may affect other queries |
| Update partition metadata | YELLOW | Modifies catalog metadata |
| Convert data to columnar format | YELLOW | Creates new data — does not delete original |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Resolution involves changing Glue catalog shared across multiple applications
- IAM permission changes affect cross-account access

## Data Sensitivity

- Query results may contain sensitive data — ensure result S3 location has appropriate encryption and access controls
- Glue catalog metadata may reveal table structures containing PII columns
- CloudTrail logs may contain query text with sensitive filter values
- S3 data sampling during diagnosis may expose sensitive content — use LIMIT and column selection

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER modify S3 bucket policies without understanding cross-account implications
- NEVER share query execution IDs or result locations that may contain sensitive data in logs
- NEVER suggest disabling workgroup query result encryption

## Phase 3 — Rollback

- If Glue table definition was modified: restore previous column definitions using `aws glue update-table` with the original schema
- If IAM permissions were changed: revert IAM policy to previous version using policy versioning
- If partition metadata was updated incorrectly: remove bad partitions with `ALTER TABLE DROP PARTITION`
- If data format conversion created bad data: delete the new table/location and restore the original table definition
- Document all changes made during remediation for audit trail

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
