---
title: "G1 — Cross-Region Aggregation"
description: "Diagnose Security Hub cross-region finding aggregation issues"
status: active
severity: HIGH
triggers:
  - "cross-region aggregation"
  - "cross-region findings"
  - "finding aggregator"
  - "regional findings"
  - "centralized view"
owner: devops-agent
objective: "Identify and resolve cross-region finding aggregation configuration issues"
context: "Security Hub findings are regional by default. Cross-region aggregation requires creating a finding aggregator in the aggregation region. The aggregator links specific regions or all regions. Cross-region aggregation is separate from cross-account aggregation."
---

## Phase 1 — Triage

MUST:
- Check for existing finding aggregators: `aws securityhub list-finding-aggregators`
- Get aggregator details if exists: `aws securityhub get-finding-aggregator --finding-aggregator-arn <aggregator-arn>`
- Verify Security Hub is enabled in the aggregation region: `aws securityhub describe-hub`
- Check which regions are linked to the aggregator
- Verify findings from linked regions appear: `aws securityhub get-findings --filters '{"Region":[{"Value":"<linked-region>","Comparison":"EQUALS"}]}' --max-results 10`

SHOULD:
- Verify Security Hub is enabled in the linked regions
- Check if the aggregator links all regions or specific regions
- Verify the aggregation region has the latest findings from linked regions

MAY:
- Check CloudTrail for aggregator configuration events
- Verify cross-region aggregation does not conflict with cross-account setup

## Phase 2 — Remediate

MUST:
- Create a finding aggregator if none exists: `aws securityhub create-finding-aggregator --region-linking-mode ALL_REGIONS`
- Or link specific regions: `aws securityhub create-finding-aggregator --region-linking-mode SPECIFIED_REGIONS --regions <region-1> <region-2>`
- Verify findings from linked regions appear in the aggregation region

SHOULD:
- Enable Security Hub in all regions that should be aggregated
- Use ALL_REGIONS mode for comprehensive coverage

MAY:
- Set up monitoring for aggregation status
- Create insights for cross-region finding analysis

## Common Issues

- symptoms: "No findings from other regions visible"
  diagnosis: "No finding aggregator is configured, or the aggregator does not link the expected regions."
  resolution: "Create a finding aggregator with ALL_REGIONS or add the missing regions."

- symptoms: "Finding aggregator exists but some regions missing"
  diagnosis: "The aggregator uses SPECIFIED_REGIONS mode and the missing regions are not listed."
  resolution: "Update the aggregator to include the missing regions or switch to ALL_REGIONS mode."

- symptoms: "Cross-region findings appear with delay"
  diagnosis: "Cross-region aggregation has eventual consistency. There is a normal propagation delay."
  resolution: "Allow time for finding propagation. Delays of minutes to hours are normal."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create finding aggregator | YELLOW | New resource; can be deleted |
| Update linked regions | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Cross-region aggregation changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-finding-aggregators` | LOW | Aggregator configuration |
| `get-finding-aggregator` | LOW | Linked regions |
| `get-findings` (cross-region) | HIGH | Security findings from other regions |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Aggregator creation | Delete via `delete-finding-aggregator` |
| Region linking change | Update via `update-finding-aggregator` |

## Output Format

```yaml
root_cause: "cross_region_aggregation — <specific_cause>"
evidence:
  - type: aggregator
    content: "<finding aggregator configuration>"
  - type: linked_regions
    content: "<linked regions and their status>"
  - type: findings
    content: "<cross-region finding visibility>"
severity: HIGH
mitigation:
  immediate: "Create or update the finding aggregator"
  long_term: "Use ALL_REGIONS mode and monitor aggregation"
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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
