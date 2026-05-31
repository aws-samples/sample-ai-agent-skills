---
title: "E2 — Subscription Filtering"
description: "Diagnose AppSync subscription filtering and event delivery issues"
status: active
severity: MEDIUM
triggers:
  - "subscription filter"
  - "subscription not receiving"
  - "enhanced subscriptions"
  - "subscription events missing"
  - "filtered subscription"
owner: devops-agent
objective: "Identify and resolve subscription filtering and event delivery issues"
context: "AppSync subscriptions can filter events server-side using enhanced subscriptions. Without filtering, all subscribers receive all events for a subscription field. Filtering requires enhanced subscriptions to be enabled. Common issues include filtering not enabled, incorrect filter expressions, missing subscription arguments, and @aws_subscribe directive misconfiguration."
---

## Phase 1 — Triage

MUST:
- Check if enhanced subscriptions are enabled: `aws appsync get-graphql-api --api-id <api-id>`
- Get the subscription resolver: `aws appsync get-resolver --api-id <api-id> --type-name Subscription --field-name <field>`
- Verify the @aws_subscribe directive references the correct mutation(s)
- Check CloudWatch Logs for subscription event delivery
- Verify the subscription filter expression in the resolver response mapping

SHOULD:
- Check if the mutation resolver returns all fields needed by the subscription filter
- Verify the subscription arguments match the filter expression
- Test with a simple subscription (no filter) to confirm basic connectivity works

MAY:
- Check subscription connection count metrics
- Verify the mutation is actually triggering the subscription (check mutation response)

## Phase 2 — Remediate

MUST:
- Enable enhanced subscriptions if server-side filtering is needed
- Ensure @aws_subscribe references the correct mutation field names
- Fix filter expressions in the subscription resolver response mapping

SHOULD:
- Ensure mutations return all fields that subscription filters reference
- Use subscription arguments for dynamic filtering
- Test subscriptions with the AppSync console's subscription tester

MAY:
- Implement client-side filtering as a fallback for complex filter logic
- Use multiple subscription fields for different filter patterns

## Common Issues

- symptoms: "Subscriber receives all events, not just filtered ones"
  diagnosis: "Enhanced subscriptions not enabled, or filter expression not in the resolver."
  resolution: "Enable enhanced subscriptions and add filter logic to the subscription resolver response mapping."

- symptoms: "Subscription never receives any events"
  diagnosis: "@aws_subscribe directive references wrong mutation name, or mutation doesn't trigger subscription."
  resolution: "Verify @aws_subscribe(mutations: ['correctMutationName']) matches the actual mutation field."

- symptoms: "Filter expression error in logs"
  diagnosis: "Filter expression references a field not returned by the mutation resolver."
  resolution: "Ensure the mutation resolver returns all fields used in the subscription filter."

## Output Format

```yaml
root_cause: "subscription_filtering — <specific_cause>"
evidence:
  - type: subscription_config
    content: "<subscription resolver and @aws_subscribe directive>"
  - type: filter_expression
    content: "<filter logic in response mapping>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter expression or enable enhanced subscriptions"
  long_term: "Add subscription integration tests to CI/CD"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable enhanced subscriptions | YELLOW | API configuration change |
| Ensure @aws_subscribe references correct mutations | YELLOW | Schema change |
| Fix filter expressions | YELLOW | Resolver code change |
| Ensure mutations return needed fields | YELLOW | Resolver code change |
| Test with AppSync console | GREEN | Non-disruptive testing |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Subscription config**: Resolver and directive configuration — INTERNAL
- **Filter expressions**: Server-side filtering logic — INTERNAL
- **Event data**: Subscription event payloads — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER disable subscription filtering without understanding data exposure impact
- NEVER expose unfiltered subscription events containing other users' data

## Phase 3 — Rollback

1. Revert subscription resolver filter expressions
2. Revert @aws_subscribe directive changes
3. Revert mutation resolver changes if they were modified
4. Disable enhanced subscriptions if it was newly enabled and caused issues
5. Verify subscriptions are receiving correct filtered events

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
