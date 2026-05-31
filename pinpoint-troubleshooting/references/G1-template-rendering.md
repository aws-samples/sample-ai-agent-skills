---
title: "G1 — Template Rendering Issues"
description: "Diagnose message template rendering and variable substitution problems"
status: active
severity: MEDIUM
triggers:
  - "template not rendering"
  - "variables not replaced"
  - "template error"
  - "message shows raw variables"
owner: devops-agent
objective: "Identify and resolve template rendering issues"
context: "Pinpoint message templates support variable substitution using {{Attribute.Name}} syntax. Variables are replaced with endpoint attributes at send time. Missing attributes result in blank values or default text."
---

## Phase 1 — Triage

MUST:
- Get template: `aws pinpoint get-email-template --template-name <name> --version <version>`
- Check template variables match endpoint attributes
- Verify endpoint has required attributes: `aws pinpoint get-endpoint --application-id <app-id> --endpoint-id <endpoint-id>`
- List template versions: `aws pinpoint list-template-versions --template-name <name> --template-type EMAIL`

SHOULD:
- Test template rendering with sample endpoint
- Check for typos in variable names (case-sensitive)
- Verify default values are set for optional variables

MAY:
- Review template across all channels (email, SMS, push)
- Check template quota limits

## Phase 2 — Remediate

MUST:
- Fix variable names to match endpoint attribute names exactly
- Set default values for variables that may be missing
- Ensure template version matches campaign/journey reference

SHOULD:
- Use consistent attribute naming across endpoints
- Test templates before campaign launch

MAY:
- Implement template validation in CI/CD pipeline

## Common Issues

- symptoms: "Message shows {{User.Name}} instead of actual name"
  diagnosis: "Variable name doesn't match endpoint attribute or attribute is missing."
  resolution: "Use correct attribute path. Set default value: {{User.Name|default:'Customer'}}."

- symptoms: "Template renders differently than preview"
  diagnosis: "Preview uses sample data, actual send uses endpoint attributes."
  resolution: "Verify endpoints have all required attributes populated."

## Output Format

```yaml
root_cause: "template_rendering — <specific_cause>"
evidence:
  - type: template_config
    content: "<template content and variables>"
  - type: endpoint_attributes
    content: "<endpoint attribute values>"
severity: MEDIUM
mitigation:
  immediate: "Fix variable names or set defaults"
  long_term: "Implement template testing with real endpoint data"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — templates may contain personalization tokens
- Service Disruption Risk: LOW — rendering issues affect message appearance
- Compliance Impact: MEDIUM — template content must comply with messaging regulations

## Escalation Conditions
- Template rendering fails for all messages
- Personalization tokens not resolving correctly
- Template rendering produces incorrect content
- Template size exceeds limits
- Template rendering causes delivery delays

## Data Sensitivity
- Template content — INTERNAL
- Personalization tokens — INTERNAL
- Customer attribute references — CONFIDENTIAL
- Template versions — INTERNAL
- Never expose customer data through template rendering errors

## Prohibited Actions
- Do NOT deploy templates without testing personalization
- Do NOT include sensitive data in template defaults
- Do NOT bypass template content review
- Do NOT use templates with broken personalization in production
- Do NOT ignore template rendering errors
- Do NOT share template configurations externally

## Phase 3 — Rollback
- Restore previous template versions
- Revert personalization token changes
- Verify template rendering after changes
- Restore default values for personalization tokens
- Document template issues and rendering errors
- Update template management procedures

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
