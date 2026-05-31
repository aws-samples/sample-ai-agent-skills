---
title: "C3 — Target Apply Errors"
description: "Diagnose errors when DMS applies changes to the target database"
status: active
severity: HIGH
triggers:
  - "target apply error"
  - "duplicate key"
  - "constraint violation"
  - "foreign key error"
  - "apply failed"
owner: devops-agent
objective: "Resolve errors that occur when DMS writes data to the target database"
context: "Target apply errors occur during full load or CDC when DMS cannot insert, update, or delete rows on the target. Common causes include primary key violations, foreign key constraints, unique index conflicts, trigger interference, or insufficient target capacity."
---

## Phase 1 — Triage

MUST:
- Check task error messages: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Check table statistics for error counts: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[*].{Table:TableName,State:TableState,Errors:ValidationFailedRecords,Inserts:Inserts,Updates:Updates}'`
- Check task settings for error handling: review ErrorBehavior settings in task JSON
- Check CDC apply latency: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencyTarget --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Review DMS task CloudWatch Logs for specific SQL errors
- Check if target has triggers that interfere with DMS operations
- Verify target table structure matches expected schema

MAY:
- Check target database for lock contention
- Review if BatchApplyEnabled is causing ordering issues

## Phase 2 — Remediate

MUST:
- For duplicate key errors: check if target already has data (use truncate-before-load)
- For foreign key errors: disable foreign keys during migration or load in correct order
- For constraint violations: fix data or modify target constraints

SHOULD:
- Disable triggers on target tables during migration
- Set appropriate error handling in task settings (suspend table vs stop task)
- Use DROP_AND_CREATE or TRUNCATE_BEFORE_LOAD target table prep mode

MAY:
- Enable BatchApplyEnabled with BatchApplyPreserveTransaction for better CDC apply
- Increase target database resources if apply is slow

## Common Issues

- symptoms: "Duplicate key violation during full load"
  diagnosis: "Target table already contains data from a previous run."
  resolution: "Use TRUNCATE_BEFORE_LOAD in target table preparation mode."

- symptoms: "Foreign key constraint violation during CDC"
  diagnosis: "CDC changes applied out of order across related tables."
  resolution: "Disable foreign keys during migration. Re-enable after cutover."

- symptoms: "Trigger error on target"
  diagnosis: "Target triggers firing on DMS inserts/updates causing errors."
  resolution: "Disable triggers during migration. Re-enable after cutover."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use TRUNCATE_BEFORE_LOAD target prep mode | RED | Deletes all target data before reload |
| Disable foreign keys during migration | YELLOW | Removes referential integrity — re-enable after |
| Disable triggers on target tables | YELLOW | Changes target behavior — re-enable after |
| Set error handling to suspend table | GREEN | Pauses table instead of stopping task |
| Enable BatchApplyEnabled for CDC | YELLOW | Changes apply behavior — may affect ordering |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Target apply errors cause data inconsistency between source and target
- Target database serves live traffic and constraint changes affect applications

## Data Sensitivity

- **Classification: HIGH**
- Error messages may contain source data values causing constraint violations
- Table statistics reveal data volumes and error rates per table
- Target schema details expose database design and constraints
- CDC apply patterns reveal data change characteristics

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest truncating target tables that serve live application traffic
- **NEVER** suggest permanently disabling foreign keys or triggers on the target

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Used TRUNCATE_BEFORE_LOAD | Cannot undo — target data was deleted before reload |
| Disabled foreign keys | Re-enable foreign keys after migration completes |
| Disabled triggers | Re-enable triggers after migration completes |
| Changed error handling to suspend table | Revert error handling settings if needed |

## Output Format

```yaml
root_cause: "target_apply — <specific_cause>"
evidence:
  - type: error_message
    content: "<specific apply error>"
  - type: table_statistics
    content: "<affected tables and error counts>"
severity: HIGH
mitigation:
  immediate: "Fix the specific apply error cause"
  long_term: "Configure proper target preparation and error handling in task settings"
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
