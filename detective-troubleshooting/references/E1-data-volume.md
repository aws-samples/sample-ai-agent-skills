---
title: "E1 — Data Volume Issues"
description: "Diagnose Detective data volume concerns including unexpected costs and ingestion limits"
status: active
severity: MEDIUM
triggers:
  - "data volume"
  - "detective cost"
  - "high data volume"
  - "ingestion volume"
  - "detective billing"
owner: devops-agent
objective: "Manage Detective data volume and associated costs"
context: "Detective pricing is based on data volume ingested. High-volume accounts with many CloudTrail events, VPC Flow Logs, or GuardDuty findings can generate significant costs. The 30-day free trial helps estimate costs."
---

## Phase 1 — Triage

MUST:
- Check data source packages enabled: `aws detective list-datasource-packages --graph-arn <graph-arn>`
- Review member account data sources: `aws detective batch-get-graph-member-datasources --graph-arn <graph-arn> --account-ids <account-ids>`
- Check which data sources contribute most volume (VPC Flow Logs is typically highest)
- Review billing in Cost Explorer for Detective charges

SHOULD:
- Identify high-volume member accounts
- Check if VPC Flow Logs data source is enabled (optional, high volume)
- Review free trial status and remaining days

MAY:
- Estimate monthly costs based on current ingestion rate
- Compare costs across regions if Detective is multi-region

## Phase 2 — Remediate

MUST:
- If costs too high, consider disabling optional data sources (VPC Flow Logs)
- Remove low-priority member accounts to reduce volume
- Monitor costs during free trial before committing

SHOULD:
- Prioritize member accounts by security risk vs. data volume
- Use Cost Explorer to track Detective spending trends

## Output Format

```yaml
root_cause: "data_volume — <specific_cause>"
evidence:
  - type: datasource_packages
    content: "<enabled data sources>"
  - type: cost_data
    content: "<Detective billing data>"
severity: MEDIUM
mitigation:
  immediate: "Disable high-volume optional data sources if needed"
  long_term: "Implement cost monitoring and member account prioritization"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Data volume information reveals security monitoring scope |
| Blast Radius | MEDIUM — Disabling data sources to reduce costs creates security blind spots |
| Rollback Complexity | LOW — Data sources can be re-enabled |
| Compliance Scope | HIGH — Cost-driven data source decisions affect security coverage |

## Escalation Conditions

- Escalate to security team before disabling any data sources for cost reasons
- Escalate to finance if Detective costs exceed budget thresholds
- Escalate to management if cost vs security coverage tradeoffs need decision

## Data Sensitivity

- **Data volume metrics** reveal security monitoring scope and scale
- **Cost data** is financial information

## PROHIBITED Actions

- Do NOT disable data sources solely for cost reasons without security team approval
- Do NOT remove member accounts to reduce costs without security risk assessment

## Phase 3 — Rollback

- Re-enable data sources that were disabled for cost management
- Re-add member accounts that were removed
- Wait 24 hours for data ingestion to resume
- Document any monitoring gaps

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
