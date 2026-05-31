---
title: "D2 — Approval Workflow Issues"
description: "Diagnose budget action approval workflow issues"
status: active
severity: HIGH
triggers: ["approval workflow", "action pending approval", "cannot approve action", "approval notification"]
owner: devops-agent
objective: "Resolve budget action approval workflow issues"
context: "Budget actions can require approval before execution. Approval requests are sent via SNS. Approvers must explicitly approve or deny via API."
---

## Phase 1 — Triage

MUST:
- Check action approval status: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify approval type is set: check ApprovalModel field (AUTOMATIC or MANUAL)
- Check SNS topic for approval notifications: `aws sns list-subscriptions-by-topic --topic-arn <topic-arn>`

SHOULD:
- Review action history for pending approvals: `aws budgets describe-budget-action-histories --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify approvers have IAM permissions for budgets:ExecuteBudgetAction

MAY:
- Check if approval notifications were delivered
- Review SNS delivery logs in CloudWatch

## Phase 2 — Remediate

MUST:
- Ensure SNS subscriptions are confirmed for approvers
- Grant budgets:ExecuteBudgetAction to approvers
- Approve or deny pending actions: `aws budgets execute-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id> --execution-type APPROVE_BUDGET_ACTION`

SHOULD:
- Set up multiple approvers for redundancy
- Configure escalation paths for unresponded approvals

MAY:
- Implement Lambda-based auto-approval for low-risk actions
- Set up Slack/Teams integration via AWS Chatbot for approval notifications

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget-action: GREEN — read-only action check"
  - "sns list-subscriptions-by-topic: GREEN — read-only listing"
  - "budgets execute-budget-action APPROVE: RED — approves potentially disruptive action"
  - "budgets execute-budget-action DENY: YELLOW — denies action, may leave cost uncontrolled"
```

## Escalation Conditions

- Approval requests not reaching approvers
- All approvers unavailable and action is time-sensitive
- Approval workflow blocks critical cost control actions
- Conflicting approvals from multiple approvers

## Data Sensitivity

- Approval notifications: MEDIUM — contain budget and action details
- Approver identities: LOW — IAM principal information
- Action details: MEDIUM — reveals cost control strategy
- SNS topic configuration: LOW — contains topic metadata

## Prohibited Actions

- NEVER auto-approve actions that stop production resources
- NEVER bypass approval workflows for convenience
- NEVER approve actions without understanding their impact
- NEVER remove approval requirements from high-impact actions

## Phase 3 — Rollback

1. Deny incorrectly approved actions if not yet executed: `aws budgets execute-budget-action --account-id <id> --budget-name <name> --action-id <id> --execution-type DENY_BUDGET_ACTION`
2. If action already executed, follow the specific action rollback procedure
3. Reset action status after rollback
4. Fix approval workflow configuration
5. Document the approval workflow issue and resolution

## Common Issues

- symptoms: "Action stuck in PENDING_APPROVAL status"
  diagnosis: "Approvers not receiving notifications or lacking permissions to approve."
  resolution: "Verify SNS subscriptions are confirmed. Grant budgets:ExecuteBudgetAction to approvers."

## Output Format

```yaml
root_cause: "approval_workflow — <specific_cause>"
evidence:
  - type: approval_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix approval workflow"
  long_term: "Set up redundant approvers and escalation paths"
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
