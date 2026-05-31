---
title: "C1 — PartiQL Query Errors"
description: "Diagnose PartiQL query syntax and execution errors"
status: active
severity: MEDIUM
triggers: ["PartiQL error", "query failed", "syntax error", "type mismatch"]
owner: devops-agent
objective: "Identify and resolve PartiQL query errors"
context: "Query errors occur when: SQL syntax used instead of PartiQL, Ion types wrong, reserved words not escaped, or document path invalid."
---

## Phase 1 — Triage

MUST:
- Check PartiQL syntax (INSERT INTO table VALUE {...}, not VALUES)
- Verify Ion data types (use backtick for timestamps: `2024-01-01T`)
- Check for reserved words (use backticks to escape)
- Verify document field paths

SHOULD:
- Test with simple queries first
- Check if WHERE clause uses indexed field
- Verify data types match between query and stored data

## Phase 2 — Remediate

MUST:
- Fix PartiQL syntax to use VALUE keyword for inserts
- Use Ion types for timestamps, decimals, blobs
- Escape reserved words with backticks

SHOULD:
- Use QLDB shell for interactive testing
- Document common PartiQL patterns

## Output Format

```yaml
root_cause: "partiql_error — <specific_cause>"
evidence:
  - type: query
    content: "<failing query>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — PartiQL errors are development concerns
- Service Disruption Risk: MEDIUM — query errors affect application functionality
- Compliance Impact: LOW — query errors are operational concerns

## Escalation Conditions
- PartiQL errors affect production applications
- Query errors cause data inconsistencies
- OCC (Optimistic Concurrency Control) conflicts increase significantly
- Transaction failures exceed normal thresholds
- PartiQL syntax changes break existing queries

## Data Sensitivity
- Query syntax — INTERNAL
- Error messages — INTERNAL (may contain data references)
- Transaction details — INTERNAL
- Table schemas — INTERNAL
- Never expose query results containing sensitive data

## Prohibited Actions
- Do NOT deploy untested PartiQL queries to production
- Do NOT ignore OCC conflict patterns
- Do NOT bypass query validation
- Do NOT run queries that bypass access controls
- Do NOT ignore transaction failure patterns
- Do NOT modify production queries without testing

## Phase 3 — Rollback
- Revert application queries to previous versions
- Restore query templates
- Verify query results after changes
- Document PartiQL issues and error patterns
- Update query development guidelines
- Re-test queries after corrections

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
