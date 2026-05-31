---
title: "B2 — Analysis Rule Issues"
description: "Diagnose Clean Rooms analysis rule configuration problems including rule types and restrictions"
status: active
severity: HIGH
triggers:
  - "analysis rule"
  - "aggregation rule"
  - "list rule"
  - "custom rule"
  - "query not allowed"
owner: devops-agent
objective: "Resolve analysis rule configuration issues and enable correct query capabilities"
context: "Analysis rules (AGGREGATION, LIST, CUSTOM) define what queries are allowed on configured tables. Each table needs at least one rule. Common issues include missing rules, overly restrictive rules, wrong rule type, and column configuration errors."
---

## Phase 1 — Triage

MUST:
- Get analysis rule: `aws cleanrooms get-configured-table-analysis-rule --configured-table-identifier <id> --analysis-rule-type <AGGREGATION|LIST|CUSTOM>`
- Verify the rule type matches the intended query pattern
- Check allowed columns, join columns, and dimension columns
- Verify aggregation thresholds if using AGGREGATION rule

SHOULD:
- Review the query that failed against the analysis rule restrictions
- Check if multiple rule types are needed (a table can have multiple)
- Verify join columns match between collaborating tables

MAY:
- Compare with a working analysis rule configuration
- Check CloudTrail for rule creation/update events

## Phase 2 — Remediate

MUST:
- Create or update analysis rule:
  ```
  aws cleanrooms create-configured-table-analysis-rule --configured-table-identifier <id> --analysis-rule-type AGGREGATION --analysis-rule-policy '{"v1":{"aggregation":{"aggregateColumns":[{"columnNames":["col1"],"function":"COUNT"}],"joinColumns":["join_col"],"dimensionColumns":["dim_col"],"scalarFunctions":["COALESCE"],"outputConstraints":[{"columnName":"col1","minimum":10,"type":"COUNT_DISTINCT"}]}}}'
  ```
- Ensure join columns are consistent across collaborating tables
- Set appropriate aggregation thresholds

SHOULD:
- Use the least permissive rule type that meets requirements
- Document analysis rule rationale for collaborators

## Output Format

```yaml
root_cause: "analysis_rule — <specific_cause>"
evidence:
  - type: rule_config
    content: "<analysis rule details>"
  - type: query_error
    content: "<query failure message>"
severity: HIGH
mitigation:
  immediate: "Fix analysis rule configuration"
  long_term: "Document analysis rules and test with sample queries"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create or update analysis rule | RED |
| Ensure join columns are consistent | YELLOW |
| Set appropriate aggregation thresholds | RED |
| Use the least permissive rule type | GREEN |
| Document analysis rule rationale | GREEN |

## Escalation Conditions

- Fix requires modifying production analysis rules
- Fix involves lowering aggregation thresholds or changing privacy constraints

## Data Sensitivity

- **CRITICAL**: analysis rules are the primary privacy enforcement mechanism controlling what queries are allowed
- Aggregation thresholds and output constraints directly protect individual record privacy

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If analysis rule was modified, restore previous rule configuration: `aws cleanrooms delete-configured-table-analysis-rule` and recreate
2. If aggregation thresholds were lowered, restore previous minimum thresholds
3. If join columns were changed, verify consistency with collaborating tables
4. Verify rollback by confirming analysis rule matches expected privacy constraints

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
