---
title: "F1 — Organization Budgets"
description: "Diagnose organization-level budget creation and management issues"
status: active
severity: MEDIUM
triggers: ["organization budget", "member account budget", "consolidated budget", "org-wide budget"]
owner: devops-agent
objective: "Resolve organization-level budget issues"
context: "Organization budgets can be created for member accounts from the management account. They require specific IAM permissions and Organizations integration."
---

## Phase 1 — Triage

MUST:
- Verify caller is management account: `aws sts get-caller-identity`
- Check Organizations status: `aws organizations describe-organization`
- List budgets for the target account: `aws budgets describe-budgets --account-id <member-account-id>`

SHOULD:
- Verify IAM permissions for cross-account budget management
- Check for SCPs affecting budget operations in member accounts
- Review linked account cost allocation: `aws ce get-cost-and-usage --time-period Start=2025-03-01,End=2025-04-01 --granularity MONTHLY --group-by Type=DIMENSION,Key=LINKED_ACCOUNT`

MAY:
- Check if consolidated billing is enabled
- Review cost allocation tags for member account filtering

## Phase 2 — Remediate

MUST:
- Use management account credentials for member account budgets
- Create budget with correct member account ID: `aws budgets create-budget --account-id <member-account-id> --budget file://budget.json`
- Ensure budget filters correctly scope to the member account

SHOULD:
- Set up budgets for all member accounts systematically
- Use CloudFormation StackSets for consistent budget deployment

MAY:
- Implement automated budget creation for new member accounts
- Set up organization-wide budget dashboards

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "organizations describe-organization: GREEN — read-only org check"
  - "budgets describe-budgets: GREEN — read-only listing"
  - "budgets create-budget (cross-account): YELLOW — creates budget in member account"
```

## Escalation Conditions

- Management account access required but not available
- Budget actions in member accounts require cross-account role setup
- Organization structure changes affect budget scope
- Consolidated billing discrepancies affect budget accuracy

## Data Sensitivity

- Member account spending: HIGH — reveals per-account cost data
- Organization structure: MEDIUM — reveals account hierarchy
- Budget configurations: MEDIUM — reveals cost governance strategy
- Cost allocation data: MEDIUM — reveals spending patterns

## Prohibited Actions

- NEVER create budgets with actions in member accounts without their awareness
- NEVER use management account for non-budget operations
- NEVER share member account cost data without authorization
- NEVER modify organization structure to fix budget issues

## Phase 3 — Rollback

1. Delete incorrectly created member account budgets: `aws budgets delete-budget --account-id <member-id> --budget-name <name>`
2. Verify no actions were triggered in member accounts
3. Revert any cross-account IAM role changes
4. Notify member account owners of budget changes
5. Document organization budget configuration

## Common Issues

- symptoms: "CreateBudget for member account returns AccessDeniedException"
  diagnosis: "Caller is not the management account or lacks budgets:CreateBudget permission."
  resolution: "Use management account credentials. Ensure IAM policy allows budgets:CreateBudget with resource scope for the member account."

## Output Format

```yaml
root_cause: "organization_budget — <specific_cause>"
evidence:
  - type: org_budget_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix organization budget configuration"
  long_term: "Automate budget deployment via StackSets"
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
