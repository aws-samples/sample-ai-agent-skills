---
title: "B2 — Index Issues"
description: "Diagnose index creation and query optimization problems"
status: active
severity: MEDIUM
triggers: ["full table scan", "index error", "slow query", "max indexes"]
owner: devops-agent
objective: "Identify and resolve index issues"
context: "Index issues occur when: max 5 indexes reached, index field wrong, or query not using index."
---

## Phase 1 — Triage

MUST:
- List indexes: execute `SELECT * FROM information_schema.user_tables` via QLDB session
- Check if query WHERE clause uses indexed field
- Verify index count (max 5 per table)
- Check if query is being rejected due to full scan

SHOULD:
- Verify index field matches query patterns
- Check if index was created before data insertion

## Phase 2 — Remediate

MUST:
- Create index on frequently queried fields: `CREATE INDEX ON table (field)`
- Ensure WHERE clause uses indexed fields
- Note: indexes cannot be dropped after creation

SHOULD:
- Plan indexes before data insertion
- Document index strategy

## Output Format

```yaml
root_cause: "index_issue — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — index management is operational
- Service Disruption Risk: MEDIUM — missing indexes cause query performance degradation
- Compliance Impact: LOW — indexing is an operational concern

## Escalation Conditions
- Index creation fails
- Missing indexes cause query timeouts
- Index limit reached preventing new indexes
- Index does not improve query performance as expected
- Multiple index issues simultaneously

## Data Sensitivity
- Index configurations — INTERNAL
- Table schemas — INTERNAL
- Query performance metrics — INTERNAL
- Index usage statistics — INTERNAL
- Never expose index configurations that reveal data structure

## Prohibited Actions
- Do NOT create indexes without understanding query patterns
- Do NOT drop indexes used by production queries
- Do NOT exceed QLDB index limits without optimization
- Do NOT ignore index-related performance warnings
- Do NOT create redundant indexes
- Do NOT modify indexes during peak usage without approval

## Phase 3 — Rollback
- Re-create dropped indexes (note: QLDB indexes cannot be dropped once created)
- Optimize queries to work with existing indexes
- Verify query performance after index changes
- Document index issues and query patterns
- Update index management guidelines
- Plan index strategy based on query analysis

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
