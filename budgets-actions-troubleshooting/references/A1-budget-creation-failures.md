---
title: "A1 — Budget Creation Failures"
description: "Diagnose budget creation and configuration failures"
status: active
severity: HIGH
triggers: ["budget creation failed", "cannot create budget", "budget error", "create-budget error"]
owner: devops-agent
objective: "Resolve budget creation failures"
context: "Budgets track cost, usage, or reservation utilization. Creation requires proper IAM permissions and valid budget parameters."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names budgets:CreateBudget`
- List existing budgets to check limits: `aws budgets describe-budgets --account-id <account-id>`
- Verify account ID is correct: `aws sts get-caller-identity`

SHOULD:
- Check CloudTrail for CreateBudget errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBudget --max-results 10`
- Verify budget name is unique within the account

MAY:
- Check if the account has reached the budget limit (default 20,000 per account)
- Review budget parameters for validation errors

## Phase 2 — Remediate

MUST:
- Fix IAM permissions — ensure budgets:CreateBudget and budgets:ModifyBudget are granted
- Use valid budget parameters (amount, time period, budget type)
- Create budget: `aws budgets create-budget --account-id <account-id> --budget file://budget.json --notifications-with-subscribers file://notifications.json`

SHOULD:
- Set up both actual and forecasted spend thresholds
- Configure SNS notifications for budget alerts

MAY:
- Use budget templates for consistent creation across accounts
- Set up cost allocation tags for filtered budgets

## Safety Ratings

```
safety_ratings:
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "budgets describe-budgets: GREEN — read-only listing"
  - "budgets create-budget: YELLOW — creates monitoring resource with potential action triggers"
  - "sts get-caller-identity: GREEN — read-only identity check"
```

## Escalation Conditions

- Budget creation fails with internal service errors after permission fixes
- Account has reached the maximum budget limit
- Budget requires cost allocation tags not yet activated
- Organization-level budget creation requires management account access

## Data Sensitivity

- `aws budgets describe-budgets`: MEDIUM — reveals spending limits and patterns
- Budget configurations: MEDIUM — reveals cost governance strategy
- Account ID: LOW — account identifier
- `aws cloudtrail lookup-events`: MEDIUM — reveals user actions

## Prohibited Actions

- NEVER create budgets with actions without understanding the action impact
- NEVER set budget amounts that do not reflect actual spending expectations
- NEVER create budgets for other accounts without proper authorization
- NEVER delete existing budgets to work around creation limits

## Phase 3 — Rollback

1. Delete incorrectly created budgets: `aws budgets delete-budget --account-id <account-id> --budget-name <name>`
2. Remove associated notifications and subscribers
3. If actions were configured, verify they have not executed
4. Verify no duplicate budgets exist after creation attempts
5. Document budget creation for governance records

## Common Issues

- symptoms: "CreateBudget returns DuplicateRecordException"
  diagnosis: "A budget with the same name already exists in the account."
  resolution: "Use a unique budget name or update the existing budget."

## Output Format

```yaml
root_cause: "budget_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix budget creation issue"
  long_term: "Automate budget creation via IaC"
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
