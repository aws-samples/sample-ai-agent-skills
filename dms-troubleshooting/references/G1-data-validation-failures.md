---
title: "G1 — Data Validation Failures"
description: "Diagnose data validation failures in DMS migrations"
status: active
severity: MEDIUM
triggers:
  - "validation failed"
  - "data mismatch"
  - "validation error"
  - "records not matching"
owner: devops-agent
objective: "Investigate and resolve data validation failures between source and target"
context: "DMS data validation compares source and target rows to verify migration accuracy. Validation can show failures due to actual data differences, timing issues during CDC, LOB columns (not validated by default), or schema differences. Transient mismatches during active CDC are common."
---

## Phase 1 — Triage

MUST:
- Check validation status: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[*].{Table:TableName,ValidationState:ValidationState,ValidationFailed:ValidationFailedRecords,ValidationSuspended:ValidationSuspendedRecords,ValidationPending:ValidationPendingRecords}'`
- Check task validation settings in task configuration
- Check if CDC is still active (transient mismatches expected): `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].{Status:Status,CDCStartPosition:CdcStartPosition}'`

SHOULD:
- Re-run validation after CDC catches up for accurate results
- Check if validation failures are on specific tables or widespread
- Verify source and target schemas match (column count, types, keys)

MAY:
- Enable validation-specific logging for detailed mismatch information
- Compare specific rows manually on source and target

## Phase 2 — Remediate

MUST:
- Wait for CDC to catch up before declaring validation failures
- Re-run validation on failed tables: stop and restart validation
- Investigate persistent mismatches by comparing source and target data

SHOULD:
- Check if LOB columns are causing mismatches (not validated by default)
- Verify primary keys exist on validated tables
- Check for data type conversion differences between source and target

MAY:
- Use custom validation queries for complex scenarios
- Enable LOB validation if LOB accuracy is critical

## Common Issues

- symptoms: "Validation shows failures during active CDC"
  diagnosis: "Source data changed between validation reads — transient mismatch."
  resolution: "Re-run validation after CDC catches up. These are usually false positives."

- symptoms: "ValidationSuspendedRecords count is high"
  diagnosis: "Validation could not compare rows due to missing keys or ongoing changes."
  resolution: "Ensure tables have primary keys. Re-run validation after CDC stabilizes."

- symptoms: "Persistent validation failures after CDC stops"
  diagnosis: "Actual data difference between source and target."
  resolution: "Compare specific rows. Check for data type conversion issues or truncation."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Re-run validation after CDC catches up | GREEN | Verification — non-destructive |
| Enable LOB validation | GREEN | Additional validation — non-destructive |
| Enable validation-specific logging | GREEN | Monitoring — non-destructive |
| Compare specific rows manually | GREEN | Investigation — read-only |
| Stop and restart validation | GREEN | Resets validation state — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Persistent validation failures indicate data loss during migration
- Validation failures block production cutover approval

## Data Sensitivity

- **Classification: HIGH**
- Validation results reveal data differences between source and target
- Row-level comparison may expose sensitive data values
- Validation failure patterns reveal data integrity issues
- Table-level validation status reveals migration accuracy per table

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest ignoring persistent validation failures in production migrations
- **NEVER** suggest disabling validation to avoid seeing failures

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Re-ran validation | No rollback needed — validation is non-destructive |
| Enabled LOB validation | Disable LOB validation if it causes performance issues |
| Enabled validation logging | Disable additional logging if no longer needed |

## Output Format

```yaml
root_cause: "validation_failure — <specific_cause>"
evidence:
  - type: validation_status
    content: "<per-table validation results>"
  - type: cdc_status
    content: "<CDC active or stopped>"
severity: MEDIUM
mitigation:
  immediate: "Re-validate after CDC catches up"
  long_term: "Include validation as standard step in migration runbook"
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
