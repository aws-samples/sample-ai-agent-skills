---
title: "C2 — Cross-Account Authorization"
description: "Diagnose cross-account authorization issues for AWS Config aggregators"
status: active
severity: MEDIUM
triggers:
  - "cross-account authorization"
  - "aggregation authorization"
  - "source account authorization"
  - "cross-account config"
  - "authorization denied"
owner: devops-agent
objective: "Resolve cross-account authorization issues preventing Config aggregator data collection"
context: "Individual account aggregators require explicit authorization from each source account. Organization aggregators bypass this but need Organizations access. Authorization issues are the most common cause of missing aggregator data."
---

## Phase 1 — Triage

MUST:
- List existing authorizations in the aggregator account: `aws configservice describe-aggregation-authorizations`
- Check aggregator source status: `aws configservice describe-configuration-aggregator-sources-status --configuration-aggregator-name <aggregator-name>`
- Verify the aggregator type: `aws configservice describe-configuration-aggregators`
- In the source account, check if authorization was granted: `aws configservice describe-aggregation-authorizations`
- Verify account IDs and regions match between aggregator and authorization

SHOULD:
- Check CloudTrail in source account for authorization events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutAggregationAuthorization --max-results 10`
- Verify the source account has Config recorder running: `aws configservice describe-configuration-recorder-status`

MAY:
- Check IAM permissions for the user/role creating authorizations
- Verify no SCP is blocking Config aggregation API calls

## Phase 2 — Remediate

MUST:
- Create missing authorizations in source accounts: `aws configservice put-aggregation-authorization --authorized-account-id <aggregator-account-id> --authorized-aws-region <aggregator-region>`
- Verify the authorization was created successfully
- Confirm data starts flowing to the aggregator

SHOULD:
- Consider switching to organization aggregator for simpler management
- Document all source account authorizations

MAY:
- Automate authorization creation using CloudFormation StackSets
- Set up monitoring for authorization status changes

## Common Issues

- symptoms: "Aggregator shows OUTDATED status for a source account"
  diagnosis: "The source account's authorization was deleted or the account/region in the authorization does not match."
  resolution: "Re-create the authorization in the source account with the correct aggregator account ID and region."

- symptoms: "Authorization exists but aggregator still shows no data"
  diagnosis: "The source account's Config recorder is stopped or not recording the expected resource types."
  resolution: "Start the Config recorder in the source account and verify resource type coverage."

- symptoms: "Cannot create authorization — AccessDeniedException"
  diagnosis: "The IAM principal lacks config:PutAggregationAuthorization permission or an SCP is blocking it."
  resolution: "Grant the required IAM permission or update the SCP to allow Config aggregation."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create aggregation authorization | YELLOW | Permission grant; reversible |
| Start Config recorder in source | YELLOW | Service enablement; reversible |

## Escalation Conditions

- Recorder configuration changes
- Cross-account authorization changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-aggregation-authorizations` | MEDIUM | Authorization details |
| `describe-configuration-aggregator-sources-status` | MEDIUM | Source account status |
| `describe-configuration-recorder-status` | LOW | Recorder status |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Authorization creation | Delete via `delete-aggregation-authorization` |
| Recorder start | Stop recorder (not recommended) |

## Output Format

```yaml
root_cause: "cross_account_authorization — <specific_cause>"
evidence:
  - type: authorizations
    content: "<aggregation authorizations in source and aggregator accounts>"
  - type: aggregator_sources
    content: "<aggregator source status>"
  - type: recorder_status
    content: "<source account recorder status>"
severity: MEDIUM
mitigation:
  immediate: "Create or fix the aggregation authorization"
  long_term: "Use organization aggregator or automate authorization management"
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
