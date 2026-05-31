---
title: "E1 — Log Insights Query Errors"
description: "Diagnose CloudWatch Logs Insights query syntax and execution errors"
status: active
severity: MEDIUM
triggers:
  - "query error"
  - "query syntax"
  - "Logs Insights"
  - "query failed"
  - "parse error"
owner: devops-agent
objective: "Identify and fix CloudWatch Logs Insights query errors"
context: "Logs Insights uses a purpose-built query language with commands: fields, filter, stats, sort, limit, parse, display. Common errors include syntax mistakes, invalid field references, incorrect regex patterns, unsupported functions, and querying log groups with no data in the time range."
---

## Phase 1 — Triage

MUST:
- Review the exact error message from the query
- Check query syntax: commands must be separated by `|` (pipe)
- Verify log group(s) exist and have data in the selected time range
- Check field names are correct (use `fields @message` to see available fields)

SHOULD:
- Verify the query uses supported functions and commands
- Check regex patterns in `parse` or `filter` commands for correctness
- Ensure `stats` aggregation functions are used correctly

MAY:
- Start with a simple query and build up complexity
- Check if the log group uses a specific log format (JSON, CLF, etc.)

## Phase 2 — Remediate

MUST:
- Fix syntax errors (missing pipes, incorrect command order)
- Fix field references (use @ prefix for system fields: @timestamp, @message, @logStream)
- Fix regex patterns in parse/filter commands

SHOULD:
- Use `fields @timestamp, @message` as a starting point to explore data
- Use `parse` to extract fields from unstructured logs before filtering
- Place `filter` commands early in the query to reduce data scanned

MAY:
- Use `display` to control which fields appear in results
- Save working queries for reuse

## Common Issues

- symptoms: "Syntax error in query"
  diagnosis: "Missing pipe separator between commands or incorrect command syntax."
  resolution: "Separate commands with |. Check command spelling (fields, filter, stats, sort, limit, parse, display)."

- symptoms: "Query returns no results"
  diagnosis: "Time range has no data, or filter condition is too restrictive."
  resolution: "Widen the time range. Start without filters and add them incrementally."

- symptoms: "Field not found in results"
  diagnosis: "Field name is incorrect or the log format doesn't contain that field."
  resolution: "Use `fields @message` to see raw log data. JSON fields are auto-discovered."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix query syntax | GREEN | Query correction, no state change |
| Fix field references | GREEN | Query correction |
| Widen time range | GREEN | Read-only query adjustment |

## Escalation Conditions

- Logs Insights queries needed for active incident investigation — escalate for priority
- Query errors affecting automated reporting — escalate to reporting team
- Log format changes breaking existing queries — escalate to application team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| Log Insights query results | HIGH | May contain application log data including PII |
| `fields @message` output | HIGH | Raw log content |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** share query results containing sensitive data without redaction

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| No state-changing actions in query fixes | No rollback needed |

## Output Format

```yaml
root_cause: "query_error — <specific_cause>"
evidence:
  - type: query
    content: "<the failing query>"
  - type: error_message
    content: "<exact error from Logs Insights>"
severity: MEDIUM
mitigation:
  immediate: "Fix query syntax, field names, or time range"
  long_term: "Document common query patterns for the team"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
