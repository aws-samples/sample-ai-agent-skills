---
title: "C2 — Data Type Mapping"
description: "Diagnose data type mapping failures between source and target"
status: active
severity: MEDIUM
triggers:
  - "data type error"
  - "type conversion failed"
  - "unsupported data type"
  - "column type mismatch"
owner: devops-agent
objective: "Resolve data type mapping issues between heterogeneous source and target databases"
context: "Heterogeneous migrations (e.g., Oracle to PostgreSQL) require data type mapping. DMS handles common mappings automatically but complex types (spatial, XML, custom) may fail. Table mapping transformation rules can override default mappings."
---

## Phase 1 — Triage

MUST:
- Check task error for type-related failures: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Check table statistics for error tables: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[?TableState==`TABLE_ERROR`].{Schema:SchemaName,Table:TableName,State:TableState}'`
- Check source and target engine types: `aws dms describe-endpoints --filters Name=endpoint-type,Values=source --query 'Endpoints[*].EngineName'`

SHOULD:
- Review table mapping rules for transformation overrides
- Check if pre-migration assessment flagged type issues: `aws dms describe-replication-task-assessment-results --replication-task-arn <task-arn>`
- Identify the specific columns causing failures from task logs

MAY:
- Run a pre-migration assessment to identify all type incompatibilities
- Check AWS documentation for supported type mappings between engines

## Phase 2 — Remediate

MUST:
- Add transformation rules in table mappings to override problematic type mappings
- Modify target schema to use compatible data types before migration
- Handle unsupported types by excluding columns or converting at source

SHOULD:
- Use pre-migration assessments to identify all type issues before starting
- Document type mapping decisions for the migration

MAY:
- Create custom conversion logic in the target using triggers or ETL
- Consider using AWS SCT for comprehensive type mapping analysis

## Common Issues

- symptoms: "Unsupported data type for column"
  diagnosis: "Source column type has no automatic mapping to target engine."
  resolution: "Add transformation rule to map to a compatible target type, or modify target schema."

- symptoms: "Data truncation on target"
  diagnosis: "Target column size smaller than source data."
  resolution: "Increase target column size or add transformation to truncate/convert."

- symptoms: "Spatial data type not supported"
  diagnosis: "DMS has limited support for spatial/geometry types."
  resolution: "Convert spatial data to WKT/WKB at source, migrate as text, convert back at target."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add transformation rules for type mappings | GREEN | Task configuration — non-destructive |
| Modify target schema for compatible types | YELLOW | Schema change on target — verify application compatibility |
| Exclude columns with unsupported types | YELLOW | Data loss for excluded columns — verify requirements |
| Run pre-migration assessment | GREEN | Assessment — non-destructive |
| Use AWS SCT for type mapping analysis | GREEN | Analysis tool — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Data type conversion causes data loss or truncation
- Target schema changes affect applications using the target database

## Data Sensitivity

- **Classification: MEDIUM**
- Source/target engine details reveal database technology stack
- Table and column names expose database schema design
- Data type mappings reveal data characteristics and constraints
- Pre-migration assessment results reveal migration complexity

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest silently truncating data without informing the user of potential data loss
- **NEVER** suggest dropping target columns to avoid type mapping issues

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Added transformation rules | Remove transformation rules and reload affected tables |
| Modified target schema | Revert target schema to previous definition |
| Excluded columns from migration | Re-add excluded columns and reload affected tables |

## Output Format

```yaml
root_cause: "data_type_mapping — <specific_cause>"
evidence:
  - type: error_tables
    content: "<tables with type errors>"
  - type: engine_pair
    content: "<source engine → target engine>"
severity: MEDIUM
mitigation:
  immediate: "Fix type mappings for failing columns"
  long_term: "Run pre-migration assessments and document type mapping strategy"
```

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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
