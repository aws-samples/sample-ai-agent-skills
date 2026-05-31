---
title: "C1 — Query Failures"
description: "Diagnose Clean Rooms query execution failures including SQL errors, rule violations, and permission issues"
status: active
severity: HIGH
triggers:
  - "query failed"
  - "protected query error"
  - "query execution"
  - "sql error clean rooms"
  - "query denied"
owner: devops-agent
objective: "Resolve query execution failures and enable successful data collaboration"
context: "Clean Rooms queries (protected queries) execute within the constraints of analysis rules. Common failures include analysis rule violations, SQL syntax errors, missing table associations, permission errors, and compute capacity issues."
---

## Phase 1 — Triage

MUST:
- Get query details: `aws cleanrooms get-protected-query --membership-identifier <membership-id> --protected-query-identifier <query-id>`
- Check query status and error message
- Verify the querying member has CAN_QUERY ability
- Verify all referenced tables have analysis rules configured
- Check table associations exist for the membership

SHOULD:
- Review the SQL query against analysis rule restrictions
- Verify join columns match between tables
- Check if aggregation thresholds are being met

MAY:
- List recent queries: `aws cleanrooms list-protected-queries --membership-identifier <membership-id>`
- Check CloudTrail for query execution events

## Phase 2 — Remediate

MUST:
- If analysis rule violation, modify query to comply with rules
- If table not associated, create the association
- If permission error, verify member abilities and IAM permissions
- If SQL error, fix query syntax

SHOULD:
- Test queries with simpler versions first
- Verify all referenced schemas: `aws cleanrooms list-schemas --collaboration-identifier <collab-id>`

## Output Format

```yaml
root_cause: "query_failure — <specific_cause>"
evidence:
  - type: query_status
    content: "<protected query details>"
  - type: error_message
    content: "<query error>"
severity: HIGH
mitigation:
  immediate: "Fix query, analysis rules, or permissions"
  long_term: "Create query templates that comply with analysis rules"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Modify query to comply with analysis rules | GREEN |
| Create table association | YELLOW |
| Verify member abilities and IAM permissions | GREEN |
| Fix SQL syntax | GREEN |
| Verify all referenced schemas | GREEN |

## Escalation Conditions

- Fix requires modifying production analysis rules to allow new query patterns
- Fix involves granting additional member abilities

## Data Sensitivity

- **HIGH**: query results contain collaborative data subject to privacy constraints
- Query patterns may reveal analytical intent and business intelligence strategies

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If table associations were created, remove incorrect ones: `aws cleanrooms delete-configured-table-association`
2. If member abilities were changed, restore previous configuration
3. If analysis rules were modified to allow queries, restore previous rule constraints
4. Verify rollback by confirming query permissions match expected access controls

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
