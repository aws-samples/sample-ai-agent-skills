---
name: budgets-actions-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Budgets Actions
  problems by analyzing budget creation, alert configuration, action
  execution, IAM policy actions, SCP actions, EC2/RDS instance actions,
  approval workflows, threshold configuration, notifications, and
  organization budgets, following structured runbooks.
  Activate when: budget creation failures, alert configuration issues,
  action execution failures, IAM policy action errors, SCP action
  failures, EC2 instance action errors, RDS instance action errors,
  approval workflow issues, threshold configuration, notification
  failures, organization budgets, or the user says something is wrong
  with Budgets Actions.
compatibility: >
  Requires AWS CLI or SDK access with budgets, iam, organizations,
  ec2, rds, sns, cloudtrail, and cloudwatch permissions.
---

# AWS Budgets Actions Diagnostics

## When to use

Any AWS Budgets Actions investigation — budget creation, alert configuration, automated action execution, IAM policy actions, SCP actions, EC2/RDS instance actions, approval workflows, threshold configuration, or notification failures.

## Investigation workflow

### Step 1 — Collect and triage

```
aws budgets describe-budgets --account-id <account-id> --query 'Budgets[*].{Name:BudgetName,Type:BudgetType,Limit:BudgetLimit,Calculated:CalculatedSpend}'
aws budgets describe-budget-actions-for-account --account-id <account-id> --query 'Actions[*].{Id:ActionId,BudgetName:BudgetName,ActionType:ActionType,Status:Status}'
aws budgets describe-notifications-for-budget --account-id <account-id> --budget-name <name>
```

### Step 2 — Domain deep dive

```
aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>
aws budgets describe-budget-action-histories --account-id <account-id> --budget-name <name> --action-id <action-id>
aws iam get-role --role-name AWSBudgetsActionsRoleName
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=budgets.amazonaws.com --max-results 20
aws budgets describe-budget-performance-history --account-id <account-id> --budget-name <name> --time-period Start=2025-01-01,End=2025-04-01
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
```

Read `references/guardrails.md` before concluding on any Budgets Actions issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `budgets describe-budgets` | List all budgets |
| `budgets describe-budget-actions-for-account` | List all actions |
| `budgets describe-budget-action` | Get action details |
| `budgets describe-budget-action-histories` | Check action history |
| `budgets describe-notifications-for-budget` | Check notifications |
| `budgets describe-budget-performance-history` | Review budget trends |
| `iam get-role` | Check action execution role |

## Gotchas: AWS Budgets Actions

- Budgets Actions can execute THREE types of actions: apply IAM policies, apply SCPs, and stop EC2/RDS instances. Each has different IAM requirements.
- Actions require a dedicated IAM role with trust policy for budgets.amazonaws.com. The role must have permissions for the specific action type.
- Actions can be set to automatic execution or require approval. Approval workflows use SNS notifications and require manual confirmation.
- Budget data has up to 24-hour delay. Actions based on current spend may trigger late because cost data is not real-time.
- SCP actions require AWS Organizations and can only be applied from the management account or delegated admin.
- EC2/RDS stop actions are NON-REVERSIBLE by Budgets. Stopped instances must be manually restarted after the budget period.
- Organization budgets can be created for member accounts from the management account but require specific IAM permissions.

## Anti-hallucination rules

1. Always cite specific budget names, action IDs, or API responses as evidence.
2. Budget cost data has up to 24-hour delay. Never claim real-time cost tracking.
3. Actions are NOT automatically reversed. Stopped instances stay stopped. Applied policies stay applied.
4. SCP actions and IAM policy actions are fundamentally different. Never conflate them.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Budget Setup | A1-A2 | Budget creation failures, alert configuration |
| B — Action Execution | B1-B2 | Action execution failures, IAM policy actions |
| C — SCP & EC2 Actions | C1-C2 | SCP action failures, EC2 instance actions |
| D — RDS & Approvals | D1-D2 | RDS instance actions, approval workflows |
| E — Thresholds & Notifications | E1-E2 | Threshold configuration, notification failures |
| F — Organization | F1 | Organization budgets |
| Z — Catch-All | Z1 | General troubleshooting |
