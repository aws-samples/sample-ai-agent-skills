---
title: "C2 — Query Syntax Issues"
description: "Diagnose PartiQL-specific syntax differences from SQL"
status: active
severity: LOW
triggers: ["INSERT VALUES", "ALTER TABLE", "JOIN error"]
owner: devops-agent
objective: "Identify and resolve PartiQL syntax issues"
---

## Phase 1 — Triage

MUST:
- Check INSERT syntax: VALUE not VALUES
- Verify UPDATE syntax uses SET with document paths
- Check if JOINs are supported for the query type
- Verify no ALTER TABLE or DDL operations unsupported by QLDB

SHOULD:
- Review PartiQL documentation for QLDB-specific syntax
- Check history function syntax for revision queries

## Phase 2 — Remediate

MUST:
- Convert SQL INSERT to PartiQL: `INSERT INTO t VALUE {'field': 'value'}`
- Use `history(table)` function for revision history
- Use `BY docId` for document ID access

## Output Format

```yaml
root_cause: "query_syntax — <specific_cause>"
severity: LOW
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — query syntax is a development concern
- Service Disruption Risk: LOW — syntax errors affect individual queries
- Compliance Impact: LOW — query syntax is an operational concern

## Escalation Conditions
- PartiQL syntax changes break existing queries
- Query syntax errors affect production applications
- Complex queries cause unexpected results
- Syntax validation inconsistencies
- Multiple query syntax issues simultaneously

## Data Sensitivity
- Query syntax and patterns — INTERNAL
- Query results — CONFIDENTIAL
- Table schemas — INTERNAL
- Application query templates — INTERNAL
- Never expose query results containing sensitive data

## Prohibited Actions
- Do NOT deploy untested queries to production
- Do NOT ignore query syntax validation errors
- Do NOT bypass query review processes
- Do NOT run queries that bypass access controls
- Do NOT share query patterns that reveal data structure
- Do NOT modify production queries without testing

## Phase 3 — Rollback
- Revert application queries to previous versions
- Restore query templates
- Verify query results after changes
- Document query syntax issues
- Update query development guidelines
- Re-test queries after syntax corrections

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
