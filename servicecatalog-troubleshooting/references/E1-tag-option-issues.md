---
title: "E1 — Tag Option Issues"
description: "Diagnose Service Catalog tag option configuration issues"
status: active
severity: MEDIUM
triggers: ["tag option", "tagging error", "tag not applied", "tag option not showing"]
owner: devops-agent
objective: "Resolve tag option configuration and association issues"
context: "Tag options enforce specific tag key-value pairs during provisioning. They are different from CloudFormation resource tags. Tag options are associated with portfolios or products."
---

## Phase 1 — Triage

MUST:
- List tag options: `aws servicecatalog list-tag-options --query 'TagOptionDetails[*].{Id:Id,Key:Key,Value:Value,Active:Active}'`
- Check tag option associations: `aws servicecatalog list-resources-for-tag-option --tag-option-id <tag-option-id>`
- Verify tag option is active (not disabled)

SHOULD:
- Check if tag option is associated with the correct portfolio or product
- Verify tag key/value comply with AWS tag naming rules

MAY:
- Review provisioned product tags to verify tag options were applied
- Check if organization tag policies conflict with tag options

## Phase 2 — Remediate

MUST:
- Create missing tag options: `aws servicecatalog create-tag-option --key <key> --value <value>`
- Associate tag option with portfolio: `aws servicecatalog associate-tag-option-with-resource --resource-id <portfolio-id> --tag-option-id <tag-option-id>`
- Reactivate disabled tag options: `aws servicecatalog update-tag-option --id <tag-option-id> --active`

SHOULD:
- Create tag options for all required compliance tags
- Associate tag options at the portfolio level for broad enforcement

MAY:
- Automate tag option management with CloudFormation
- Set up tag option libraries for consistent tagging across portfolios

## Safety Ratings

```
safety_ratings:
  - "list-tag-options: GREEN — read-only listing"
  - "create-tag-option: GREEN — creates tag option definition"
  - "associate-tag-option-with-resource: YELLOW — enforces tagging on provisioning"
  - "delete-tag-option: YELLOW — removes tag enforcement"
```

## Escalation Conditions

- Tag options conflict with organization tag policies
- Tag option changes affect multiple portfolios and products
- Compliance requirements mandate specific tag options that don't exist
- Tag options not being applied to provisioned product resources

## Data Sensitivity

- `aws servicecatalog list-tag-options`: LOW — tag key-value definitions
- `aws servicecatalog list-resources-for-tag-option`: LOW — resource associations
- Tag option values: LOW — typically contain cost center, environment, or team names
- Provisioned product tags: LOW — applied tag values

## Prohibited Actions

- NEVER delete tag options that are required for compliance
- NEVER create tag options with sensitive data in values (use references instead)
- NEVER disable tag options without understanding which portfolios are affected
- NEVER modify tag options on imported portfolios (they are read-only)

## Phase 3 — Rollback

1. If wrong tag option was created: `aws servicecatalog delete-tag-option --id <tag-option-id>` (if not associated)
2. If tag option was associated with wrong resource: `aws servicecatalog disassociate-tag-option-from-resource --resource-id <resource-id> --tag-option-id <tag-option-id>`
3. If tag option was disabled: reactivate with `--active` flag
4. Verify existing provisioned products are not affected by tag option changes
5. Document tag option changes and notify compliance team

## Common Issues

- symptoms: "Tag option is created but not showing during provisioning"
  diagnosis: "Tag option is not associated with the portfolio or product, or it is inactive."
  resolution: "Associate tag option with the portfolio: aws servicecatalog associate-tag-option-with-resource --resource-id <portfolio-id> --tag-option-id <id>."

## Output Format

```yaml
root_cause: "tag_option — <specific_cause>"
evidence:
  - type: tag_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix tag option configuration"
  long_term: "Implement tag option governance strategy aligned with compliance requirements"
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
