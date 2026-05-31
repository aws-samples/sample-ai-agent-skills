---
title: "E1 — Threshold Configuration"
description: "Diagnose budget threshold configuration issues"
status: active
severity: MEDIUM
triggers: ["threshold wrong", "threshold not triggering", "budget threshold", "percentage threshold"]
owner: devops-agent
objective: "Resolve budget threshold configuration issues"
context: "Thresholds define when notifications and actions trigger. They can be based on actual spend, forecasted spend, or percentage of budget amount."
---

## Phase 1 — Triage

MUST:
- Check budget and threshold configuration: `aws budgets describe-budget --account-id <account-id> --budget-name <name>`
- Review notification thresholds: `aws budgets describe-notifications-for-budget --account-id <account-id> --budget-name <name>`
- Check budget performance: `aws budgets describe-budget-performance-history --account-id <account-id> --budget-name <name> --time-period Start=2025-01-01,End=2025-04-01`

SHOULD:
- Verify threshold type (PERCENTAGE vs ABSOLUTE_VALUE)
- Check comparison operator (GREATER_THAN, LESS_THAN, EQUAL_TO)
- Verify notification type (ACTUAL vs FORECASTED)

MAY:
- Review historical threshold triggers
- Check if cost data delay is affecting threshold evaluation

## Phase 2 — Remediate

MUST:
- Set appropriate threshold values based on budget amount and risk tolerance
- Use correct notification type (ACTUAL for current spend, FORECASTED for predicted)
- Configure multiple thresholds for graduated response (50%, 80%, 100%)

SHOULD:
- Set forecasted thresholds for early warning: `aws budgets create-notification --account-id <id> --budget-name <name> --notification '{"NotificationType":"FORECASTED","ComparisonOperator":"GREATER_THAN","Threshold":100}' --subscribers '[{"SubscriptionType":"SNS","Address":"<arn>"}]'`
- Account for cost data delay when setting thresholds

MAY:
- Use absolute value thresholds for specific dollar amounts
- Implement dynamic thresholds based on historical patterns

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget: GREEN — read-only budget check"
  - "budgets describe-notifications-for-budget: GREEN — read-only listing"
  - "budgets describe-budget-performance-history: GREEN — read-only history"
  - "budgets create-notification: YELLOW — adds notification trigger"
```

## Escalation Conditions

- Thresholds not triggering despite spending exceeding the budget
- Forecasted thresholds producing false positives
- Multiple thresholds triggering simultaneously causing action conflicts
- Threshold configuration unclear for complex filtered budgets

## Data Sensitivity

- Budget amounts: MEDIUM — reveals spending limits
- Threshold values: LOW — contains percentage or dollar values
- Performance history: MEDIUM — reveals spending patterns
- Notification configuration: LOW — contains threshold metadata

## Prohibited Actions

- NEVER set only 100% actual threshold (too late for preventive action)
- NEVER remove all thresholds from budgets with active actions
- NEVER set thresholds without understanding the budget period (monthly vs annual)
- NEVER ignore forecasted threshold alerts — they provide early warning

## Phase 3 — Rollback

1. Delete incorrectly configured notifications: `aws budgets delete-notification --account-id <id> --budget-name <name> --notification <notification-json>`
2. Recreate notifications with correct thresholds
3. Verify actions are linked to the correct thresholds
4. Test threshold triggering with current budget data
5. Document threshold configuration rationale

## Common Issues

- symptoms: "Budget at 90% but 80% threshold notification not received"
  diagnosis: "Notification type set to FORECASTED instead of ACTUAL, or SNS subscription not confirmed."
  resolution: "Verify notification type is ACTUAL for current spend alerts. Confirm SNS subscriptions."

## Output Format

```yaml
root_cause: "threshold_configuration — <specific_cause>"
evidence:
  - type: threshold_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix threshold configuration"
  long_term: "Implement graduated thresholds with multiple notification channels"
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
