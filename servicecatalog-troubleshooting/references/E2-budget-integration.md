---
title: "E2 — Budget Integration"
description: "Diagnose Service Catalog budget integration issues"
status: active
severity: MEDIUM
triggers: ["budget", "cost tracking", "budget association", "spending limit"]
owner: devops-agent
objective: "Resolve Service Catalog budget association and tracking issues"
context: "Service Catalog integrates with AWS Budgets to track costs of provisioned products. Budgets are associated with portfolios or products."
---

## Phase 1 — Triage

MUST:
- List budgets associated with portfolio: `aws servicecatalog list-budgets-for-resource --resource-id <portfolio-id>`
- Check budget details: `aws budgets describe-budget --account-id <account-id> --budget-name <budget-name>`
- Verify budget exists in AWS Budgets service

SHOULD:
- Check if budget notifications are configured: `aws budgets describe-notifications-for-budget --account-id <account-id> --budget-name <budget-name>`
- Verify cost allocation tags are enabled for tracking

MAY:
- Review actual vs budgeted costs: `aws budgets describe-budget --account-id <account-id> --budget-name <budget-name> --query 'Budget.{Limit:BudgetLimit,Actual:CalculatedSpend}'`
- Check if provisioned products are tagged for cost tracking

## Phase 2 — Remediate

MUST:
- Create budget if missing: `aws budgets create-budget --account-id <account-id> --budget <budget-json>`
- Associate budget with portfolio: `aws servicecatalog associate-budget-with-resource --budget-name <budget-name> --resource-id <portfolio-id>`
- Ensure cost allocation tags are activated in Billing console

SHOULD:
- Set up budget alerts for threshold notifications
- Configure SNS notifications for budget alerts

MAY:
- Create separate budgets per product for granular cost tracking
- Automate budget creation as part of portfolio provisioning

## Safety Ratings

```
safety_ratings:
  - "list-budgets-for-resource: GREEN — read-only listing"
  - "describe-budget: GREEN — read-only budget details"
  - "associate-budget-with-resource: GREEN — creates association, no cost impact"
  - "create-budget: GREEN — creates budget tracking, no spending impact"
```

## Escalation Conditions

- Budget creation requires billing account access
- Cost allocation tags need to be activated by account administrator
- Budget alerts not triggering despite threshold being exceeded
- Cross-account budget tracking requires consolidated billing setup

## Data Sensitivity

- `aws budgets describe-budget`: MEDIUM — reveals spending amounts
- `aws servicecatalog list-budgets-for-resource`: LOW — budget associations
- Budget notifications: LOW — threshold configuration
- Cost allocation data: MEDIUM — reveals resource spending patterns

## Prohibited Actions

- NEVER delete budgets without understanding which portfolios they track
- NEVER modify budget limits without finance team approval
- NEVER disable budget alerts during active spending periods
- NEVER share budget details with unauthorized personnel

## Phase 3 — Rollback

1. If budget was associated with wrong resource: `aws servicecatalog disassociate-budget-from-resource --budget-name <name> --resource-id <resource-id>`
2. If budget was created with wrong limits: `aws budgets update-budget --account-id <account-id> --new-budget <corrected-budget-json>`
3. If budget was deleted: recreate with original configuration
4. Verify budget tracking is restored after rollback
5. Document budget changes and notify finance team

## Common Issues

- symptoms: "Budget shows $0 actual spend despite active provisioned products"
  diagnosis: "Cost allocation tags are not activated, or provisioned products are not tagged for cost tracking."
  resolution: "Activate cost allocation tags in Billing console. Ensure provisioned products have tags matching the budget filter."

## Output Format

```yaml
root_cause: "budget_integration — <specific_cause>"
evidence:
  - type: budget_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix budget association"
  long_term: "Implement automated budget tracking for all portfolios"
```

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
