---
title: "B1 — Action Execution Failures"
description: "Diagnose budget action execution failures"
status: active
severity: CRITICAL
triggers: ["action not executing", "action failed", "budget action error", "action stuck"]
owner: devops-agent
objective: "Resolve budget action execution failures"
context: "Budget actions execute when thresholds are breached. Failures can stem from IAM role issues, missing permissions, or target resource problems."
---

## Phase 1 — Triage

MUST:
- Check action status: `aws budgets describe-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Review action history: `aws budgets describe-budget-action-histories --account-id <account-id> --budget-name <name> --action-id <action-id>`
- Verify execution role: `aws iam get-role --role-name <role-name>`

SHOULD:
- Check role trust policy allows budgets.amazonaws.com
- Verify role has permissions for the action type
- Review CloudTrail for action execution events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=budgets.amazonaws.com --max-results 20`

MAY:
- Check if the action requires approval and is pending
- Verify target resources exist and are in a valid state

## Phase 2 — Remediate

MUST:
- Fix IAM execution role trust policy to allow budgets.amazonaws.com
- Grant required permissions to the execution role based on action type
- Verify action threshold has been breached (check budget performance)

SHOULD:
- Test action execution manually: `aws budgets execute-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id> --execution-type APPROVE_BUDGET_ACTION`
- Set up CloudWatch alarms for action execution failures

MAY:
- Implement retry logic for transient failures
- Set up SNS notifications for action status changes

## Safety Ratings

```
safety_ratings:
  - "budgets describe-budget-action: GREEN — read-only action check"
  - "budgets describe-budget-action-histories: GREEN — read-only history"
  - "iam get-role: GREEN — read-only role check"
  - "budgets execute-budget-action: RED — manually triggers budget action"
```

## Escalation Conditions

- Action execution fails repeatedly with permission errors after role fixes
- Action targets production resources and manual execution is needed
- Multiple actions failing simultaneously across budgets
- Action execution role compromised or deleted

## Data Sensitivity

- Action configuration: MEDIUM — reveals cost control strategy
- Action history: MEDIUM — reveals execution timeline and targets
- IAM role details: MEDIUM — reveals permission scope
- `aws cloudtrail lookup-events`: MEDIUM — reveals execution details

## Prohibited Actions

- NEVER manually execute actions on production resources without approval
- NEVER modify the execution role to have overly broad permissions
- NEVER ignore action execution failures — they indicate cost control gaps
- NEVER delete action history for audit purposes

## Phase 3 — Rollback

1. If action executed incorrectly, reverse the specific action (restart instances, detach policies)
2. Reset action status: `aws budgets execute-budget-action --account-id <account-id> --budget-name <name> --action-id <action-id> --execution-type RESET_BUDGET_ACTION`
3. Fix the execution role before re-enabling the action
4. Verify target resources are in expected state after rollback
5. Document the execution failure and resolution

## Common Issues

- symptoms: "Action status shows EXECUTION_FAILURE"
  diagnosis: "Execution role missing required permissions or trust policy incorrect."
  resolution: "Update role trust policy to include budgets.amazonaws.com. Add required permissions for the action type."

## Output Format

```yaml
root_cause: "action_execution — <specific_cause>"
evidence:
  - type: execution_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix action execution"
  long_term: "Implement action monitoring and alerting"
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
