---
title: "D1 — Template Errors"
description: "Diagnose SES email template creation and rendering errors"
status: active
severity: MEDIUM
triggers:
  - "template error"
  - "template not found"
  - "template rendering failed"
  - "invalid template"
owner: devops-agent
objective: "Resolve SES email template issues"
context: "SES templates use Handlebars-style syntax for personalization. Templates have a 500KB size limit. Template names are case-sensitive. Templates are region-specific and must exist in the sending region."
---

## Phase 1 — Triage

MUST:
- Verify template exists: `aws sesv2 get-email-template --template-name <name>`
- Check template syntax for valid Handlebars expressions
- Verify template is in the correct region
- Test template rendering: `aws sesv2 test-render-email-template --template-name <name> --template-data '{"key":"value"}'`

SHOULD:
- Check template size is within 500KB limit
- Verify all referenced variables are provided in template data
- Review template HTML for rendering issues

MAY:
- List all templates: `aws sesv2 list-email-templates`
- Compare template versions if recently updated

## Phase 2 — Remediate

MUST:
- Fix Handlebars syntax errors in template
- Ensure all referenced variables are provided in send call
- Create template in the correct region

SHOULD:
- Test template rendering before production use
- Use default values for optional variables: `{{#if name}}{{name}}{{else}}Customer{{/if}}`

MAY:
- Implement template versioning strategy
- Set up template validation in CI/CD pipeline

## Common Issues

- symptoms: "TemplateDoesNotExist error"
  diagnosis: "Template name is case-sensitive or template is in a different region."
  resolution: "Verify exact template name and region. Template names are case-sensitive."

- symptoms: "Template rendering fails with missing variable"
  diagnosis: "Template references a variable not provided in template data."
  resolution: "Provide all required variables or add conditional logic for optional variables."

- symptoms: "Template HTML renders incorrectly"
  diagnosis: "HTML syntax issues or email client compatibility problems."
  resolution: "Validate HTML. Test across email clients. Use inline CSS for compatibility."

## Output Format

```yaml
root_cause: "template_error — <specific_cause>"
evidence:
  - type: template_config
    content: "<template details and syntax>"
  - type: render_test
    content: "<test render results>"
severity: MEDIUM
mitigation:
  immediate: "Fix template syntax or provide missing variables"
  long_term: "Implement template testing and validation pipeline"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Email content may contain customer PII and business communications
- Recipient lists contain personal email addresses
- Bounce and complaint data reveals delivery issues with specific addresses
- Classification: HIGHLY CONFIDENTIAL - email content and recipient PII

## Prohibited Actions

- NEVER send test emails to real customer addresses
- NEVER modify production DKIM/SPF/DMARC records without email team approval
- NEVER disable bounce or complaint handling in production
- NEVER remove addresses from the suppression list without investigating the original cause

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling DKIM/SPF/DMARC"
  - "NEVER suggest removing sending limits"
  - "NEVER suggest disabling bounce/complaint handling"
