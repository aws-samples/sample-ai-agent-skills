---
title: "C1 — Aggregator Issues"
description: "Diagnose AWS Config aggregator data collection and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "aggregator not working"
  - "aggregator no data"
  - "aggregator missing accounts"
  - "config aggregator error"
  - "aggregator stale data"
owner: devops-agent
objective: "Identify and resolve Config aggregator issues preventing centralized compliance visibility"
context: "Config aggregators collect compliance data from multiple accounts and regions. Issues arise from missing authorizations, incorrect aggregator type (individual vs organization), stale data due to eventual consistency, or source account recorder problems."
---

## Phase 1 — Triage

MUST:
- Describe aggregators: `aws configservice describe-configuration-aggregators`
- Check aggregator sources: `aws configservice describe-configuration-aggregator-sources-status --configuration-aggregator-name <aggregator-name>`
- List aggregation authorizations: `aws configservice describe-aggregation-authorizations`
- Verify the aggregator type (ORGANIZATION or ACCOUNT)
- Check if source accounts have Config recorder running

SHOULD:
- Get aggregated compliance by rule: `aws configservice describe-aggregate-compliance-by-config-rules --configuration-aggregator-name <aggregator-name>`
- Check for pending authorizations from source accounts
- Verify Organizations access if using organization aggregator: `aws organizations describe-organization`

MAY:
- Check CloudTrail for aggregator-related events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutConfigurationAggregator --max-results 10`
- Verify network connectivity if using cross-region aggregation

## Phase 2 — Remediate

MUST:
- Add missing aggregation authorizations from source accounts
- Ensure source accounts have Config recorder running
- Verify the aggregator configuration matches the intended scope

SHOULD:
- Switch to organization aggregator if managing all accounts in an organization
- Re-create the aggregator if the configuration is fundamentally wrong

MAY:
- Set up monitoring for aggregator source status
- Document the expected source accounts and regions

## Common Issues

- symptoms: "Aggregator shows no data from specific accounts"
  diagnosis: "Those accounts have not authorized the aggregator account, or their Config recorder is stopped."
  resolution: "Run put-aggregation-authorization in each source account, or use organization aggregator."

- symptoms: "Aggregator data is stale or delayed"
  diagnosis: "Aggregator data is eventually consistent. There is a normal delay of up to several hours."
  resolution: "Wait for data propagation. If persistently stale, check source account recorder and delivery channel status."

- symptoms: "Organization aggregator shows AccessDeniedException"
  diagnosis: "The account is not the management account or delegated admin, or Organizations trusted access is not enabled."
  resolution: "Use management account or delegated admin. Enable trusted access for Config."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add aggregation authorizations | YELLOW | Permission grant; reversible |
| Switch to organization aggregator | YELLOW | Aggregator change; reversible |
| Re-create aggregator | YELLOW | New resource; old can be deleted |

## Escalation Conditions

- Recorder configuration changes
- Cross-account authorization changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-aggregators` | LOW | Aggregator configuration |
| `describe-configuration-aggregator-sources-status` | MEDIUM | Source account status |
| `describe-aggregation-authorizations` | MEDIUM | Authorization details |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Authorization addition | Delete via `delete-aggregation-authorization` |
| Aggregator creation | Delete via `delete-configuration-aggregator` |

## Output Format

```yaml
root_cause: "aggregator_issue — <specific_cause>"
evidence:
  - type: aggregator_config
    content: "<describe-configuration-aggregators output>"
  - type: source_status
    content: "<aggregator sources status>"
  - type: authorizations
    content: "<aggregation authorizations>"
severity: MEDIUM
mitigation:
  immediate: "Fix authorization or source account issues"
  long_term: "Monitor aggregator source status and consider organization aggregator"
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
