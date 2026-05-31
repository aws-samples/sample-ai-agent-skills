


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Optimize resolver data source operations | YELLOW | Resolver/data source change |
| Enable AppSync caching | YELLOW | Cache infrastructure change |
| Optimize DynamoDB queries | YELLOW | Query pattern change |
| Use provisioned concurrency for Lambda | YELLOW | Lambda scaling change |
| Implement batch resolvers | YELLOW | Resolver architecture change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Latency metrics**: Query performance data — INTERNAL
- **Resolver configuration**: Data source and resolver details — INTERNAL
- **Query patterns**: Slow query identification — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER add full table scans to resolve latency issues
- NEVER disable field-level logging to reduce latency

## Phase 3 — Rollback

1. Revert resolver code changes to previous version
2. Revert DynamoDB index or capacity changes
3. Revert Lambda provisioned concurrency settings
4. Disable caching if it was newly enabled and caused issues
5. Monitor latency metrics to confirm rollback resolved the issue

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
