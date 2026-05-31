---
title: "F2 — StackSet Constraints"
description: "Diagnose Service Catalog StackSet constraint configuration issues"
status: active
severity: HIGH
triggers: ["StackSet constraint", "multi-account deployment", "StackSet failed", "cross-account provisioning"]
owner: devops-agent
objective: "Resolve StackSet constraint configuration and multi-account deployment issues"
context: "StackSet constraints enable multi-account/multi-region deployment via CloudFormation StackSets. Requires specific IAM roles in admin and target accounts."
---

## Phase 1 — Triage

MUST:
- List StackSet constraints: `aws servicecatalog list-constraints-for-portfolio --portfolio-id <portfolio-id> --query 'ConstraintDetails[?Type==`STACKSET`]'`
- Check constraint details: `aws servicecatalog describe-constraint --id <constraint-id>`
- Verify admin role exists: `aws iam get-role --role-name AWSCloudFormationStackSetAdministrationRole`

SHOULD:
- Verify execution role exists in target accounts: `aws iam get-role --role-name AWSCloudFormationStackSetExecutionRole`
- Check StackSet status: `aws cloudformation list-stack-sets --query 'Summaries[?Status==`ACTIVE`]'`

MAY:
- Review StackSet operation history: `aws cloudformation list-stack-set-operations --stack-set-name <name>`
- Check target account permissions for resource creation

## Phase 2 — Remediate

MUST:
- Create admin role if missing: AWSCloudFormationStackSetAdministrationRole with cloudformation.amazonaws.com trust
- Create execution role in target accounts: AWSCloudFormationStackSetExecutionRole with admin role trust
- Create StackSet constraint: `aws servicecatalog create-constraint --portfolio-id <id> --product-id <id> --type STACKSET --parameters '{"AccountList":["<accounts>"],"RegionList":["<regions>"],"AdminRole":"arn:aws:iam::<account>:role/AWSCloudFormationStackSetAdministrationRole","ExecutionRole":"AWSCloudFormationStackSetExecutionRole"}'`

SHOULD:
- Test StackSet deployment in a subset of accounts first
- Verify network connectivity between admin and target accounts

MAY:
- Use service-managed StackSets with Organizations for automatic deployment
- Set up failure tolerance and max concurrent operations

## Safety Ratings

```
safety_ratings:
  - "list-constraints-for-portfolio: GREEN — read-only listing"
  - "create-constraint (STACKSET): YELLOW — enables multi-account deployment"
  - "StackSet deployment: RED — creates resources in multiple accounts"
  - "delete-constraint (STACKSET): RED — may orphan resources in target accounts"
```

## Escalation Conditions

- Execution role missing in target accounts requires cross-account IAM changes
- StackSet operations fail in specific accounts due to SCPs
- Admin role permissions insufficient for StackSet operations
- Multi-region deployment fails due to regional service availability

## Data Sensitivity

- `aws servicecatalog describe-constraint`: MEDIUM — contains account lists and role ARNs
- `aws cloudformation list-stack-set-operations`: MEDIUM — deployment history
- `aws iam get-role`: MEDIUM — role configuration
- Target account list: MEDIUM — reveals organization structure

## Prohibited Actions

- NEVER deploy StackSets to production accounts without testing
- NEVER use AdministratorAccess for the execution role
- NEVER delete StackSet constraints without cleaning up deployed stack instances
- NEVER modify admin or execution roles without understanding cross-account impact

## Phase 3 — Rollback

1. If StackSet was deployed to wrong accounts: delete stack instances from those accounts
2. If constraint was created with wrong configuration: update or delete and recreate
3. If execution role was modified: restore original permissions in all target accounts
4. If StackSet operations failed partially: review per-account status and fix individually
5. Document multi-account changes and notify all affected account owners

## Common Issues

- symptoms: "StackSet constraint provisioning fails with 'StackSet operation failed in account <id>'"
  diagnosis: "AWSCloudFormationStackSetExecutionRole is missing or has insufficient permissions in the target account."
  resolution: "Create the execution role in the target account with trust policy allowing the admin role to assume it."

## Output Format

```yaml
root_cause: "stackset_constraint — <specific_cause>"
evidence:
  - type: stackset_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix StackSet constraint configuration"
  long_term: "Use service-managed StackSets with Organizations for automatic role management"
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
