---
title: "E1 — Budget Configuration"
description: "Diagnose budget creation, tracking, and enforcement issues"
status: active
severity: MEDIUM
triggers: ["budget error", "budget not tracking", "spending exceeded", "cannot create budget", "budget blocking jobs"]
owner: devops-agent
objective: "Resolve budget configuration and tracking issues"
context: "Budgets control spending by blocking new job submissions when limits are reached. They do NOT stop running jobs. Budgets track approximate costs based on worker usage."
---

## Phase 1 — Triage

MUST:
- List budgets: `aws deadline list-budgets --farm-id <farm-id> --query 'budgets[*].{BudgetId:budgetId,DisplayName:displayName,Status:status,ApproximateDollarLimit:approximateDollarLimit}'`
- Get budget details: `aws deadline get-budget --farm-id <farm-id> --budget-id <budget-id>`
- Check budget usage vs limit

SHOULD:
- Verify budget schedule (start/end dates)
- Check which queues are associated with the budget
- Review budget actions configuration

MAY:
- Check CloudWatch metrics for budget utilization
- Review historical budget consumption patterns

## Phase 2 — Remediate

MUST:
- Fix budget limit if too low for workload requirements
- Ensure budget schedule covers the intended time period
- Associate budget with correct queues

SHOULD:
- Configure budget actions (notifications) for threshold alerts
- Set appropriate budget limits based on historical usage

MAY:
- Create separate budgets for different projects or teams
- Implement budget approval workflows

## Safety Ratings

```
safety_ratings:
  - "list-budgets: GREEN — read-only budget enumeration"
  - "get-budget: GREEN — read-only budget details"
  - "create-budget: YELLOW — creates spending control"
  - "update-budget: YELLOW — changes spending limits"
  - "delete-budget: YELLOW — removes spending control, allows unlimited submission"
```

## Escalation Conditions

- Budget exhausted blocking production render submissions
- Budget tracking shows inaccurate costs requiring investigation
- Budget increase requires management approval
- Multiple teams affected by shared budget exhaustion

## Data Sensitivity

- Budget configuration: MEDIUM — reveals spending limits and project budgets
- Budget usage: MEDIUM — reveals actual spending patterns
- Queue associations: LOW — budget-queue mapping
- Budget actions: LOW — notification configuration

## Prohibited Actions

- NEVER delete budgets on production farms without replacement controls
- NEVER set budget limits to unlimited without management approval
- NEVER assume budgets stop running jobs — they only block new submissions
- NEVER share budget details outside the authorized finance/operations team

## Phase 3 — Rollback

1. If budget limit was changed incorrectly, revert: `aws deadline update-budget --farm-id <id> --budget-id <id>`
2. If budget was deleted, recreate with original configuration
3. If budget actions were modified, restore previous action configuration
4. Verify job submission is unblocked after budget fix

## Common Issues

- symptoms: "Job submission blocked by budget but spending seems within limit"
  diagnosis: "Budget tracking is approximate and may include pending job costs."
  resolution: "Check approximate usage including pending jobs. Increase budget limit if needed."

- symptoms: "Budget shows $0 usage despite running jobs"
  diagnosis: "Budget not associated with the queue where jobs are running."
  resolution: "Associate budget with the correct queue. Verify queue-budget association."

## Output Format

```yaml
root_cause: "budget_configuration — <specific_cause>"
evidence:
  - type: budget_detail
    content: "<budget configuration and usage>"
severity: MEDIUM
mitigation:
  immediate: "Adjust budget limit or associations"
  long_term: "Implement budget monitoring and forecasting"
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
