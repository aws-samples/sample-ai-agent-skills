---
title: "G2 — Row Count Mismatch"
description: "Diagnose row count mismatches between source and target after DMS migration"
status: active
severity: MEDIUM
triggers:
  - "row count mismatch"
  - "missing rows"
  - "extra rows"
  - "count different"
owner: devops-agent
objective: "Identify why source and target row counts differ and resolve the discrepancy"
context: "Row count mismatches can occur due to table mapping filters, failed rows, CDC timing, LOB errors, duplicate key handling, or task error settings that skip rows. Check table statistics for inserts, updates, deletes, and error counts."
---

## Phase 1 — Triage

MUST:
- Check table statistics: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[*].{Schema:SchemaName,Table:TableName,FullLoadRows:FullLoadRows,Inserts:Inserts,Updates:Updates,Deletes:Deletes,DDLs:Ddls,FullLoadCondtnlChkFailedRows:FullLoadCondtnlChkFailedRows,FullLoadErrorRows:FullLoadErrorRows}'`
- Check for error rows in table statistics
- Check task error handling settings (ErrorBehavior — suspend table vs skip row)
- Verify table mapping selection rules include all expected tables

SHOULD:
- Compare source row count with target row count for specific tables
- Check if CDC deletes account for the difference
- Verify no transformation rules are filtering rows

MAY:
- Check DMS task logs for skipped or errored rows
- Run data validation for detailed row-level comparison

## Phase 2 — Remediate

MUST:
- Investigate error rows — fix the cause and reload affected tables
- If rows were skipped due to error handling, fix errors and re-migrate
- Reload specific tables: `aws dms reload-tables --replication-task-arn <task-arn> --tables-to-reload SchemaName=<schema>,TableName=<table>`

SHOULD:
- Change error handling from skip to suspend to catch future issues
- Run data validation after reload to confirm counts match
- Document the root cause of the mismatch

MAY:
- Use custom queries to identify specific missing rows
- Set up monitoring for error row counts

## Common Issues

- symptoms: "Target has fewer rows than source"
  diagnosis: "Rows failed during migration and were skipped by error handling."
  resolution: "Check FullLoadErrorRows. Fix errors. Reload affected tables."

- symptoms: "Target has more rows than source"
  diagnosis: "Source deletes during CDC not applied, or duplicate inserts."
  resolution: "Check CDC delete counts. Verify primary keys exist for proper CDC apply."

- symptoms: "Counts match for some tables but not others"
  diagnosis: "Specific tables had errors or were partially loaded."
  resolution: "Check per-table statistics. Reload tables with mismatches."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Reload specific tables | YELLOW | Reloads data — may cause temporary inconsistency |
| Change error handling from skip to suspend | GREEN | Improves error visibility — non-destructive |
| Run data validation after reload | GREEN | Verification — non-destructive |
| Fix error rows and re-migrate | YELLOW | Data correction — verify fix before applying |
| Set up monitoring for error row counts | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Row count mismatch indicates data loss during migration
- Mismatch blocks production cutover approval

## Data Sensitivity

- **Classification: HIGH**
- Table statistics reveal row counts and data volumes per table
- Error row details may contain sensitive data values
- Row count comparisons expose data integrity between source and target
- Error handling settings reveal migration risk tolerance

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest ignoring row count mismatches in production migrations
- **NEVER** suggest deleting target data to force a count match

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Reloaded specific tables | Re-reload if reload introduced new issues |
| Changed error handling settings | Revert error handling to previous configuration |
| Fixed and re-migrated error rows | Verify fix did not introduce new mismatches |

## Output Format

```yaml
root_cause: "row_count_mismatch — <specific_cause>"
evidence:
  - type: table_statistics
    content: "<per-table row counts and error counts>"
  - type: error_handling
    content: "<task error behavior settings>"
severity: MEDIUM
mitigation:
  immediate: "Reload affected tables and verify counts"
  long_term: "Use strict error handling and data validation in all migrations"
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
