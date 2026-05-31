---
title: "B1 — Resource Configuration Errors"
description: "Diagnose resource configuration errors on the Application Composer canvas"
status: active
severity: MEDIUM
triggers: ["resource configuration error", "invalid property", "resource card error"]
owner: devops-agent
objective: "Resolve resource property configuration issues in Application Composer"
context: "Resource property panels are schema-driven. Not all CloudFormation properties are exposed in the visual editor."
---

## Phase 1 — Triage

MUST:
- Validate the current template: `sam validate --template-file template.yaml --lint`
- Check the specific resource configuration in template source
- Verify resource type is supported with enhanced card: review Application Composer documentation

SHOULD:
- Compare resource properties against CloudFormation resource specification
- Check for conflicting property combinations

MAY:
- Review CloudFormation resource schema: `aws cloudformation describe-type --type RESOURCE --type-name <resource-type>`

## Phase 2 — Remediate

MUST:
- Fix invalid property values in the template source
- Remove conflicting property combinations
- Ensure required properties are present for each resource

SHOULD:
- Use template source view for properties not exposed in the visual editor
- Validate after each change: `sam validate --template-file template.yaml --lint`

MAY:
- Switch to template source editing for complex configurations
- Reference CloudFormation documentation for property constraints

## Safety Ratings

```
safety_ratings:
  - "sam validate: GREEN — read-only template validation"
  - "cloudformation describe-type: GREEN — read-only schema retrieval"
  - "template source editing: YELLOW — modifies template file"
```

## Escalation Conditions

- Resource configuration panel shows incorrect property types or constraints
- Valid CloudFormation properties are rejected by Application Composer
- Resource card fails to render despite valid template syntax
- Configuration changes cause cascade errors in connected resources

## Data Sensitivity

- `sam validate`: LOW — validates template structure
- `aws cloudformation describe-type`: LOW — retrieves public resource schema
- Template content: MEDIUM — may contain configuration values and resource identifiers

## Prohibited Actions

- NEVER delete a resource to fix a configuration error without backing up the template
- NEVER change resource logical IDs as this causes CloudFormation replacement
- NEVER modify resource properties in production stacks without change set review
- NEVER assume Application Composer property panels show all available properties

## Phase 3 — Rollback

1. Restore template from version control if configuration changes caused errors: `git checkout -- template.yaml`
2. If a resource was accidentally modified, use CloudFormation drift detection: `aws cloudformation detect-stack-drift --stack-name <name>`
3. Revert to the last working template version and re-apply changes incrementally
4. Validate the restored template: `sam validate --template-file template.yaml --lint`
5. Document which properties required manual template editing vs visual editor

## Common Issues

- symptoms: "Property field is missing from the resource configuration panel"
  diagnosis: "Not all CloudFormation properties are exposed in the visual editor."
  resolution: "Switch to template source view and add the property manually in YAML/JSON."

## Output Format

```yaml
root_cause: "resource_configuration — <specific_cause>"
evidence:
  - type: config_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix resource properties"
  long_term: "Use template source for advanced configurations"
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
