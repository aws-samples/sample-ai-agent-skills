---
title: "C1 — Launch Constraint Errors"
description: "Diagnose Service Catalog launch constraint configuration errors"
status: active
severity: HIGH
triggers: ["launch constraint", "provisioning role", "role not authorized", "constraint error"]
owner: devops-agent
objective: "Resolve launch constraint configuration and permission issues"
context: "Launch constraints specify the IAM role for provisioning. Without them, end user permissions are used. The role must have permissions for all template resources."
---

## Phase 1 — Triage

MUST:
- List constraints for the portfolio: `aws servicecatalog list-constraints-for-portfolio --portfolio-id <portfolio-id> --query 'ConstraintDetails[?Type==`LAUNCH`]'`
- Check constraint details: `aws servicecatalog describe-constraint --id <constraint-id>`
- Verify the launch role exists and has correct permissions: `aws iam get-role --role-name <role-name>`

SHOULD:
- Simulate launch role permissions against required actions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names cloudformation:CreateStack ec2:RunInstances`
- Check role trust policy allows servicecatalog.amazonaws.com to assume it

MAY:
- Review CloudTrail for AssumeRole failures by Service Catalog: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 20`
- Compare launch role permissions with template resource requirements

## Phase 2 — Remediate

MUST:
- Create launch constraint if missing: `aws servicecatalog create-constraint --portfolio-id <portfolio-id> --product-id <product-id> --type LAUNCH --parameters '{"RoleArn":"arn:aws:iam::<account>:role/<role>"}'`
- Update role trust policy to allow Service Catalog: add servicecatalog.amazonaws.com as trusted principal
- Grant the launch role permissions for all resources in the product template

SHOULD:
- Use least-privilege permissions for the launch role
- Test provisioning after constraint changes

MAY:
- Create separate launch roles per product for fine-grained access
- Use IAM policy conditions to restrict resource creation scope

## Safety Ratings

```
safety_ratings:
  - "list-constraints-for-portfolio: GREEN — read-only constraint listing"
  - "create-constraint: YELLOW — creates new constraint affecting provisioning behavior"
  - "update-constraint: YELLOW — modifies provisioning behavior"
  - "delete-constraint: RED — removes constraint, changes provisioning permissions model"
```

## Escalation Conditions

- Launch role requires permissions that security team must approve
- Trust policy changes require IAM admin access
- Constraint creation fails due to portfolio being imported (read-only)
- Multiple products need different launch roles with complex permission sets

## Data Sensitivity

- `aws servicecatalog describe-constraint`: MEDIUM — contains role ARN
- `aws iam get-role`: MEDIUM — role configuration and trust policy
- `aws iam simulate-principal-policy`: LOW — permission simulation
- `aws cloudtrail lookup-events`: MEDIUM — authentication records

## Prohibited Actions

- NEVER use AdministratorAccess for launch constraint roles
- NEVER delete launch constraints without understanding the impact on end users
- NEVER modify launch roles used by multiple products without testing all products
- NEVER add wildcard resource permissions to launch roles

## Phase 3 — Rollback

1. If constraint was created with wrong role: `aws servicecatalog update-constraint --id <constraint-id> --parameters '{"RoleArn":"<correct-role-arn>"}'`
2. If constraint was deleted: recreate with original role ARN
3. If role permissions were modified: restore original policy version
4. If trust policy was changed: restore original trust policy document
5. Test provisioning after rollback to verify functionality

## Common Issues

- symptoms: "Provisioning fails with 'Service Catalog cannot assume role arn:aws:iam::...'"
  diagnosis: "Launch role trust policy does not include servicecatalog.amazonaws.com as a trusted principal."
  resolution: "Update role trust policy to allow servicecatalog.amazonaws.com to assume the role."

## Output Format

```yaml
root_cause: "launch_constraint — <specific_cause>"
evidence:
  - type: constraint_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix launch constraint configuration"
  long_term: "Standardize launch roles with least-privilege templates"
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
