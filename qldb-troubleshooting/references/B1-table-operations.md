---
title: "B1 — Table Operation Issues"
description: "Diagnose table creation and management problems"
status: active
severity: MEDIUM
triggers: ["table creation failed", "table not found", "table error"]
owner: devops-agent
objective: "Identify and resolve table operation issues"
context: "Table issues occur when: PartiQL syntax wrong, permissions insufficient, or table already exists."
---

## Phase 1 — Triage

MUST:
- List tables: execute `SELECT * FROM information_schema.user_tables` via QLDB session
- Verify PartiQL CREATE TABLE syntax
- Check IAM permissions for qldb:SendCommand
- Verify ledger is ACTIVE

SHOULD:
- Check if table name conflicts with existing table
- Verify table name follows naming rules

## Phase 2 — Remediate

MUST:
- Fix PartiQL syntax: `CREATE TABLE tableName`
- Grant qldb:SendCommand permission
- Ensure ledger is ACTIVE

## Output Format

```yaml
root_cause: "table_operation — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — table operations involve ledger data
- Service Disruption Risk: MEDIUM — table issues affect application data access
- Compliance Impact: HIGH — QLDB tables contain immutable audit data

## Escalation Conditions
- Table creation fails
- Table operations cause ledger performance degradation
- Table data integrity issues detected
- Table operations blocked by permissions
- Multiple table operations fail simultaneously

## Data Sensitivity
- Table data — CONFIDENTIAL
- Table schemas — INTERNAL
- Table operation logs — INTERNAL
- Data integrity verification — INTERNAL
- Never expose table data to unauthorized parties

## Prohibited Actions
- Do NOT drop tables containing production data without backup verification
- Do NOT modify table structures without understanding application impact
- Do NOT bypass table operation permissions
- Do NOT ignore table operation errors
- Do NOT create tables without proper indexing
- Do NOT perform bulk table operations without monitoring

## Phase 3 — Rollback
- QLDB tables are immutable — dropped tables cannot be recovered
- Restore data from journal exports if needed
- Revert application changes that caused table issues
- Verify data integrity after table operations
- Document table operation issues
- Update table management procedures

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
