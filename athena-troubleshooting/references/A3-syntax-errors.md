---
title: "A3 — Syntax Errors"
description: "Diagnose and resolve Athena SQL syntax errors"
status: active
severity: MEDIUM
triggers:
  - "syntax error"
  - "SYNTAX_ERROR"
  - "mismatched input"
  - "unexpected token"
  - "line X:Y"
owner: devops-agent
objective: "Identify the syntax error cause and provide the correct Athena SQL syntax"
context: "Athena engine v3 uses Trino SQL syntax and engine v2 uses Presto SQL syntax. Syntax differences between versions and from standard SQL cause common errors."
---

## Phase 1 — Triage

MUST:
- Get the error message: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.Status.StateChangeReason'`
- Check the engine version: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.EngineVersion'`
- Identify the query text: `aws athena get-query-execution --query-execution-id <query-id> --query 'QueryExecution.Query'`
- Verify table and column names exist: `aws glue get-table --database-name <db> --name <table>`

SHOULD:
- Check if the function exists in the engine version (v2 vs v3 function differences)
- Verify data types in the query match column types in the table
- Check for reserved word conflicts with column or table names

MAY:
- Test simplified version of the query to isolate the syntax issue
- Check Athena documentation for engine-specific syntax

## Phase 2 — Remediate

MUST:
- If engine version mismatch: adjust syntax for the target engine version
- If reserved word conflict: quote identifiers with double quotes
- If function not available: use the equivalent function for the engine version
- If type mismatch: add explicit CAST() expressions

SHOULD:
- Use ANSI SQL syntax where possible for cross-engine compatibility
- Quote all identifiers that might conflict with reserved words

MAY:
- Consider upgrading workgroup to engine v3 for broader SQL support
- Use prepared statements for parameterized queries

## Common Issues

- symptoms: "SYNTAX_ERROR: line 1:42: mismatched input 'MERGE'"
  diagnosis: "MERGE statement used on engine v2 which does not support it."
  resolution: "Upgrade to engine v3 or use INSERT INTO with CTAS pattern instead."

- symptoms: "COLUMN_NOT_FOUND with correct column name"
  diagnosis: "Column name is a reserved word (e.g., date, time, user, order)."
  resolution: "Quote the column name with double quotes: SELECT \"date\" FROM table."

- symptoms: "Function not registered: try_cast"
  diagnosis: "Engine v2 does not support try_cast function."
  resolution: "Use CAST with TRY or upgrade to engine v3."

## Output Format

```yaml
root_cause: "Syntax error — <specific_cause>"
evidence:
  - type: error_message
    content: "<syntax error details>"
  - type: engine_version
    content: "<v2 or v3>"
severity: MEDIUM
mitigation:
  immediate: "Fix the SQL syntax for the target engine version"
  long_term: "Standardize on engine v3 and use ANSI SQL where possible"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Adjust syntax for target engine version | GREEN | Query modification only |
| Quote identifiers with double quotes | GREEN | Query modification only |
| Use equivalent function for engine version | GREEN | Query modification only |
| Add explicit CAST() expressions | GREEN | Query modification only |
| Upgrade workgroup to engine v3 | YELLOW | Affects all queries in workgroup — test first |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Engine version upgrade may change query behavior for existing workloads
- Syntax changes affect shared queries or views

## Data Sensitivity

- Query text may contain sensitive filter values or table references
- Error messages may reveal table structure or column names containing sensitive data
- Engine version changes may affect query result precision for numeric types

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER upgrade engine version in production without testing in a separate workgroup first
- NEVER modify shared views or saved queries without stakeholder approval

## Phase 3 — Rollback

- If workgroup engine version was upgraded: revert using `aws athena update-work-group` with previous engine version
- If shared views were modified: restore original view definitions using `CREATE OR REPLACE VIEW`
- Document all workgroup changes for audit trail

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
