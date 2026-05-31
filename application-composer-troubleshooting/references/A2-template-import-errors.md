---
title: "A2 — Template Import Errors"
description: "Diagnose template import failures in Application Composer"
status: active
severity: MEDIUM
triggers: ["template import failed", "cannot import template", "import error"]
owner: devops-agent
objective: "Resolve template import issues in Application Composer"
context: "Application Composer imports SAM/CloudFormation templates but may silently drop unsupported resources or properties."
---

## Phase 1 — Triage

MUST:
- Validate template syntax: `aws cloudformation validate-template --template-body file://template.yaml`
- Check template size: `wc -c template.yaml` (must be under 51,200 bytes for direct upload)
- Verify YAML/JSON is well-formed: `python3 -c "import yaml; print(yaml.safe_load(open('template.yaml')))"`

SHOULD:
- Count resources in template: `grep -c 'AWS::' template.yaml`
- Check for unsupported intrinsic functions or custom macros

MAY:
- Compare resource types against Application Composer supported list
- Review template for nested stacks or cross-stack references

## Phase 2 — Remediate

MUST:
- Fix YAML/JSON syntax errors identified by validation
- Remove or simplify unsupported intrinsic function patterns
- Ensure Transform section is correct for SAM templates

SHOULD:
- Split large templates exceeding resource limits
- Document which resources were dropped during import

MAY:
- Convert between SAM and CloudFormation format if needed
- Simplify complex Condition/Mapping sections

## Safety Ratings

```
safety_ratings:
  - "cloudformation validate-template: GREEN — read-only syntax validation"
  - "wc -c: GREEN — read-only file size check"
  - "python3 yaml.safe_load: GREEN — read-only YAML parsing"
  - "grep: GREEN — read-only text search"
```

## Escalation Conditions

- Valid templates consistently fail to import with no error message
- Import succeeds but critical resources are silently dropped
- Template uses CloudFormation features that Application Composer cannot represent
- Import causes browser crash or Application Composer UI freeze

## Data Sensitivity

- `aws cloudformation validate-template`: LOW — validates syntax only, no resource creation
- `python3 yaml.safe_load`: LOW — local file parsing
- Template content: MEDIUM — may contain resource ARNs, account IDs, or configuration details

## Prohibited Actions

- NEVER suggest removing resources from the template just to make import work
- NEVER modify the original template without creating a backup first
- NEVER assume dropped resources are unimportant
- NEVER import production templates into Application Composer without reviewing what may be lost

## Phase 3 — Rollback

1. Keep the original template file as backup before any import attempts
2. If import modified the template, restore from backup: `cp template.yaml.bak template.yaml`
3. If resources were dropped, manually re-add them to the template source
4. Validate the restored template: `aws cloudformation validate-template --template-body file://template.yaml`
5. Document which resources or properties are not supported by Application Composer

## Common Issues

- symptoms: "Import succeeds but some resources are missing from the canvas"
  diagnosis: "Application Composer silently drops unsupported resource types."
  resolution: "Check template source — resources exist but are not rendered. Edit them in the template source view."

## Output Format

```yaml
root_cause: "template_import — <specific_cause>"
evidence:
  - type: import_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix template syntax or format"
  long_term: "Maintain Application Composer-compatible template structure"
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
