---
title: "F1 — Advanced Query Errors"
description: "Diagnose AWS Config advanced query syntax and execution errors"
status: active
severity: MEDIUM
triggers:
  - "query error"
  - "select resource config"
  - "query syntax error"
  - "advanced query failed"
  - "SQL query config"
owner: devops-agent
objective: "Identify and resolve AWS Config advanced query errors"
context: "Config advanced queries use a SQL-like syntax to query current resource configurations. Errors arise from syntax issues, unsupported fields, incorrect resource type names, or permission problems. Queries only work against the current configuration snapshot."
---

## Phase 1 — Triage

MUST:
- Attempt the query and capture the error: `aws configservice select-resource-config --expression "<query>"`
- Verify the resource type name is correct (e.g., AWS::EC2::Instance)
- Check that queried fields are valid for the resource type
- Verify IAM permissions include config:SelectResourceConfig
- Confirm the recorder is recording the queried resource type

SHOULD:
- Start with a simple query to verify basic functionality: `aws configservice select-resource-config --expression "SELECT resourceId, resourceType WHERE resourceType = 'AWS::EC2::Instance'"`
- Check if the resource type has any recorded configuration items
- Review the query for SQL syntax issues (missing quotes, incorrect operators)

MAY:
- Check AWS documentation for supported query fields per resource type
- Test with the Config console query editor for syntax validation

## Phase 2 — Remediate

MUST:
- Fix query syntax errors (proper quoting, valid operators, correct field names)
- Use supported resource type names in the exact AWS format
- Ensure the queried resource type is being recorded

SHOULD:
- Use the Config console query editor for syntax validation before CLI
- Break complex queries into simpler parts for debugging

MAY:
- Save frequently used queries as stored queries: `aws configservice put-stored-query --stored-query <query-config>`
- Document valid query patterns for the team

## Common Issues

- symptoms: "ValidationException: The expression is not valid"
  diagnosis: "SQL syntax error — missing quotes around string values, unsupported operator, or invalid field name."
  resolution: "Wrap string values in single quotes. Use supported operators (=, LIKE, IN). Verify field names."

- symptoms: "Query returns empty results but resources exist"
  diagnosis: "The resource type is not being recorded, or the WHERE clause is too restrictive."
  resolution: "Verify the recorder covers the resource type. Simplify the WHERE clause and add conditions incrementally."

- symptoms: "AccessDeniedException on select-resource-config"
  diagnosis: "The IAM principal lacks config:SelectResourceConfig permission."
  resolution: "Add config:SelectResourceConfig to the IAM policy."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix query syntax | GREEN | Query correction; no state change |
| Ensure resource type is recorded | YELLOW | Recorder config change; reversible |

## Escalation Conditions

- Recorder configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `select-resource-config` | MEDIUM | Resource configuration data |
| `get-discovered-resource-counts` | LOW | Aggregate resource counts |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Query fix | N/A — query correction only |
| Recorder config change | Revert recorder configuration |

## Output Format

```yaml
root_cause: "query_error — <specific_cause>"
evidence:
  - type: query
    content: "<the query that failed>"
  - type: error
    content: "<error message>"
  - type: recorder_config
    content: "<recorded resource types>"
severity: MEDIUM
mitigation:
  immediate: "Fix the query syntax or field references"
  long_term: "Document valid query patterns and use stored queries"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
