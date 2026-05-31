---
title: "C2 — Template Constraint Issues"
description: "Diagnose Service Catalog template constraint configuration issues"
status: active
severity: MEDIUM
triggers: ["template constraint", "parameter restriction", "constraint rules", "parameter not allowed"]
owner: devops-agent
objective: "Resolve template constraint configuration and rules syntax issues"
context: "Template constraints restrict CloudFormation parameter values using JSON rules syntax. They are different from launch constraints and use CloudFormation rules format."
---

## Phase 1 — Triage

MUST:
- List template constraints: `aws servicecatalog list-constraints-for-portfolio --portfolio-id <portfolio-id> --query 'ConstraintDetails[?Type==`TEMPLATE`]'`
- Check constraint details and rules: `aws servicecatalog describe-constraint --id <constraint-id>`
- Verify rules syntax matches CloudFormation rules format

SHOULD:
- Compare constraint rules with product template parameters
- Check if constraint references valid parameter names and values

MAY:
- Test provisioning with different parameter values to identify which constraint blocks
- Review CloudTrail for constraint-related provisioning failures

## Phase 2 — Remediate

MUST:
- Fix rules syntax to use proper CloudFormation rules format: `{"Rules":{"Rule1":{"Assertions":[{"Assert":{"Fn::Contains":[["t2.micro","t3.micro"],"InstanceType"]},"AssertDescription":"Only t2.micro and t3.micro allowed"}]}}}`
- Ensure parameter names in rules match template parameter names exactly
- Update constraint: `aws servicecatalog update-constraint --id <constraint-id> --parameters '<rules-json>'`

SHOULD:
- Validate rules JSON before applying
- Test constraint with a provisioning dry run

MAY:
- Create multiple template constraints for different restriction categories
- Document constraint rules for portfolio consumers

## Safety Ratings

```
safety_ratings:
  - "list-constraints-for-portfolio: GREEN — read-only listing"
  - "describe-constraint: GREEN — read-only constraint details"
  - "update-constraint: YELLOW — modifies parameter restrictions"
  - "delete-constraint: YELLOW — removes parameter restrictions"
```

## Escalation Conditions

- Complex rules syntax cannot express the required restriction
- Constraint conflicts with other constraints on the same product
- Template parameters changed but constraints were not updated
- Constraint blocks legitimate provisioning requests

## Data Sensitivity

- `aws servicecatalog describe-constraint`: LOW — constraint rules
- `aws servicecatalog list-constraints-for-portfolio`: LOW — constraint metadata
- Constraint rules content: LOW — parameter restriction definitions
- `aws cloudtrail lookup-events`: MEDIUM — user activity records

## Prohibited Actions

- NEVER use IAM policy syntax for template constraints (use CloudFormation rules syntax)
- NEVER delete template constraints without understanding which parameters they restrict
- NEVER create overly restrictive constraints that block all valid parameter combinations
- NEVER modify constraints on imported portfolios (they are read-only)

## Phase 3 — Rollback

1. If constraint rules were broken: restore previous rules JSON
2. If constraint was deleted: recreate with original rules
3. If constraint blocks valid provisioning: temporarily remove and recreate with corrected rules
4. Verify provisioning works after rollback with expected parameter values
5. Document constraint changes and notify portfolio consumers

## Common Issues

- symptoms: "Provisioning fails with 'Parameter value is not allowed by constraint' but the value looks correct"
  diagnosis: "Template constraint rules syntax is incorrect or references wrong parameter name."
  resolution: "Review constraint rules JSON. Ensure parameter names match exactly. Use CloudFormation rules syntax, not IAM policy syntax."

## Output Format

```yaml
root_cause: "template_constraint — <specific_cause>"
evidence:
  - type: constraint_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix template constraint rules"
  long_term: "Validate constraint rules as part of product version CI/CD"
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
