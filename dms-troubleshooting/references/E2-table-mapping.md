---
title: "E2 — Table Mapping"
description: "Diagnose table mapping configuration issues in DMS tasks"
status: active
severity: MEDIUM
triggers:
  - "table mapping error"
  - "wrong tables migrated"
  - "tables missing"
  - "transformation rule error"
  - "selection rule"
owner: devops-agent
objective: "Fix table mapping rules to correctly select and transform tables for migration"
context: "Table mappings control which tables DMS migrates and how they are transformed. Selection rules include/exclude tables. Transformation rules rename schemas, tables, or columns. Rules are evaluated in order. Wildcards use % not *."
---

## Phase 1 — Triage

MUST:
- Check current table mappings: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].TableMappings'`
- Check table statistics to see what was actually selected: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[*].{Schema:SchemaName,Table:TableName,State:TableState}'`
- Verify task error messages: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`

SHOULD:
- Validate JSON syntax of table mapping rules
- Check rule ordering (first match wins for selection rules)
- Verify wildcard patterns use % (not *)

MAY:
- Test with a minimal table mapping to isolate the issue
- Compare expected vs actual table list

## Phase 2 — Remediate

MUST:
- Fix selection rules to include correct schemas and tables
- Use % as wildcard character (not *)
- Ensure rule ordering is correct (exclude rules before include-all)

SHOULD:
- Modify task with corrected mappings: `aws dms modify-replication-task --replication-task-arn <task-arn> --table-mappings file://table-mappings.json`
- Test with a dry run or single table before full migration
- Add transformation rules for schema/table renaming if needed

MAY:
- Use selection rules with explicit include for critical tables
- Add column-level transformations for specific requirements

## Common Issues

- symptoms: "No tables selected for migration"
  diagnosis: "Selection rule schema or table pattern doesn't match any source objects."
  resolution: "Check schema and table names are correct. Use % for wildcards. Names may be case-sensitive."

- symptoms: "Wrong tables included"
  diagnosis: "Wildcard pattern too broad or rule ordering incorrect."
  resolution: "Add explicit exclude rules before the broad include rule."

- symptoms: "Table mapping JSON parse error"
  diagnosis: "Invalid JSON syntax in table mappings."
  resolution: "Validate JSON syntax. Check for missing commas, brackets, or quotes."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix selection rules (include/exclude patterns) | YELLOW | Changes which tables are migrated — verify scope |
| Modify task with corrected mappings | YELLOW | Task modification — requires restart to apply |
| Add transformation rules for renaming | GREEN | Mapping change — non-destructive |
| Add column-level transformations | GREEN | Mapping change — non-destructive |
| Test with single table before full migration | GREEN | Testing — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Table mapping changes affect which production tables are replicated
- Incorrect mappings caused data to be written to wrong target tables

## Data Sensitivity

- **Classification: MEDIUM**
- Table mapping rules reveal database schema and migration scope
- Selection patterns expose which tables contain critical data
- Transformation rules reveal schema differences between source and target
- Table statistics reveal actual data volumes per table

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest using overly broad wildcards that include system tables
- **NEVER** suggest removing exclude rules without understanding why they were added

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified table mapping rules | Revert to previous table mapping JSON |
| Added transformation rules | Remove added transformation rules |
| Changed selection patterns | Revert selection patterns and reload affected tables |

## Output Format

```yaml
root_cause: "table_mapping — <specific_cause>"
evidence:
  - type: table_mappings
    content: "<current table mapping rules>"
  - type: selected_tables
    content: "<tables actually selected>"
severity: MEDIUM
mitigation:
  immediate: "Fix table mapping rules"
  long_term: "Document and version-control table mappings"
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
