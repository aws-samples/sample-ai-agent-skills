---
title: "E1 — Schema Conversion"
description: "Diagnose schema conversion issues in heterogeneous DMS migrations"
status: active
severity: MEDIUM
triggers:
  - "schema conversion error"
  - "DDL conversion failed"
  - "stored procedure not converted"
  - "SCT error"
owner: devops-agent
objective: "Resolve schema conversion issues for heterogeneous database migrations"
context: "Heterogeneous migrations require schema conversion before data migration. AWS SCT handles DDL conversion. DMS can create basic tables but cannot convert stored procedures, triggers, views, or functions. Schema mismatches cause data migration failures."
---

## Phase 1 — Triage

MUST:
- Check source and target engines: `aws dms describe-endpoints --query 'Endpoints[*].{Id:EndpointIdentifier,Type:EndpointType,Engine:EngineName}'`
- Check if target schema exists and matches expectations
- Check task errors related to schema: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Run pre-migration assessment: `aws dms describe-replication-task-assessment-results --replication-task-arn <task-arn>`

SHOULD:
- Verify target table structures match source (column count, types, keys)
- Check if DMS target table preparation mode is set correctly (DO_NOTHING, DROP_AND_CREATE, TRUNCATE_BEFORE_LOAD)
- Review table mapping for schema/table name transformations

MAY:
- Use AWS SCT assessment report for comprehensive conversion analysis
- Check for unsupported features in the target engine

## Phase 2 — Remediate

MUST:
- Use AWS SCT for heterogeneous schema conversion before DMS data migration
- Fix target schema to match DMS requirements (primary keys, supported types)
- Set appropriate target table preparation mode in task settings

SHOULD:
- Convert stored procedures and functions manually or with SCT assistance
- Add primary keys to target tables (DMS performs better with PKs)
- Document manual conversion steps for objects SCT cannot convert

MAY:
- Use SCT extension packs for complex conversions
- Create target schema manually for full control over the conversion

## Common Issues

- symptoms: "Table creation failed on target"
  diagnosis: "DMS auto-create cannot handle complex source table definitions."
  resolution: "Pre-create target tables using SCT or manual DDL. Set DO_NOTHING mode."

- symptoms: "Missing stored procedures on target"
  diagnosis: "DMS does not migrate stored procedures, triggers, or views."
  resolution: "Use AWS SCT to convert and deploy stored procedures separately."

- symptoms: "Primary key required error"
  diagnosis: "Some DMS features (validation, CDC) require primary keys."
  resolution: "Add primary keys to target tables. Use unique indexes as alternatives."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use AWS SCT for schema conversion | GREEN | Analysis and conversion tool — non-destructive |
| Fix target schema (add PKs, fix types) | YELLOW | Schema change on target — verify application compatibility |
| Set target table preparation mode | YELLOW | Controls table creation behavior — verify intent |
| Convert stored procedures manually | GREEN | Manual conversion — non-destructive |
| Pre-create target tables with SCT | GREEN | Schema deployment — non-destructive if target is empty |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Schema conversion affects stored procedures used by applications
- Target schema changes impact applications already using the target database

## Data Sensitivity

- **Classification: MEDIUM**
- Source/target engine details reveal database technology stack
- Schema conversion reports expose database complexity and design
- Stored procedure code may contain business logic and data access patterns
- Table structures reveal data model and relationships

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest dropping target tables with existing data without confirmation
- **NEVER** suggest skipping stored procedure conversion without documenting the gap

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Created target schema with SCT | Drop created schema objects if conversion is incorrect |
| Modified target table structures | Revert target schema to previous definition |
| Set target table preparation mode | Change mode before restarting task if incorrect |

## Output Format

```yaml
root_cause: "schema_conversion — <specific_cause>"
evidence:
  - type: engine_pair
    content: "<source engine → target engine>"
  - type: schema_issues
    content: "<specific conversion failures>"
severity: MEDIUM
mitigation:
  immediate: "Fix target schema to support the migration"
  long_term: "Use SCT for comprehensive schema conversion before DMS"
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
