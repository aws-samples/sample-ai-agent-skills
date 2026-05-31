---
title: "E2 — Data Retention Issues"
description: "Diagnose Detective data retention questions and historical data availability"
status: active
severity: LOW
triggers:
  - "data retention"
  - "historical data"
  - "data expiry"
  - "old data detective"
  - "retention period"
owner: devops-agent
objective: "Clarify data retention policies and ensure historical data availability"
context: "Detective retains data in the behavior graph for 12 months. Historical data is automatically backfilled when new members join. Data older than 12 months is not available. Retention is not configurable."
---

## Phase 1 — Triage

MUST:
- Verify behavior graph creation date: `aws detective list-graphs`
- Check member account join date: `aws detective get-members --graph-arn <graph-arn> --account-ids <account-id>`
- Determine the time range of the data being queried
- Verify data is within the 12-month retention window

SHOULD:
- Check if the member account was recently added (backfill in progress)
- Verify data sources were continuously enabled during the retention period

MAY:
- Check if data gaps exist due to temporary data source disablement
- Review investigation time scopes against available data

## Phase 2 — Remediate

MUST:
- If data is older than 12 months, it is not available in Detective
- If member recently added, wait for historical backfill (up to several hours)
- Adjust investigation time scope to available data range

SHOULD:
- Use CloudTrail or other services for data older than 12 months
- Ensure continuous data source enablement to avoid gaps

## Output Format

```yaml
root_cause: "data_retention — <specific_cause>"
evidence:
  - type: graph_creation
    content: "<graph creation date>"
  - type: member_join_date
    content: "<member account join date>"
severity: LOW
mitigation:
  immediate: "Adjust query to available data range"
  long_term: "Maintain continuous data source coverage and plan for 12-month limit"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Retention policies affect security data availability |
| Blast Radius | LOW — Retention is fixed at 12 months; no configuration changes |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | MEDIUM — 12-month retention may not meet all compliance requirements |

## Escalation Conditions

- Escalate to compliance if 12-month retention is insufficient for regulatory requirements
- Escalate to security team if historical data is needed beyond the retention window

## Data Sensitivity

- **Retention window** defines how far back investigations can look
- **Historical data** beyond 12 months is not available in Detective

## PROHIBITED Actions

- Do NOT assume Detective retains data beyond 12 months
- Do NOT rely solely on Detective for long-term security data retention

## Phase 3 — Rollback

- No rollback needed — retention is fixed and not configurable
- For data older than 12 months, use CloudTrail, VPC Flow Logs, or other long-term storage

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
