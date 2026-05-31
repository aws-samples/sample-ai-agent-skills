---
title: "D2 — Personalization"
description: "Diagnose template personalization and bulk email issues"
status: active
severity: MEDIUM
triggers:
  - "personalization failed"
  - "variables not replaced"
  - "bulk template error"
  - "template data missing"
owner: devops-agent
objective: "Resolve template personalization issues in SES"
context: "SES supports personalization via Handlebars syntax in templates. Bulk email (SendBulkEmail) allows per-recipient personalization. Template data is JSON. Nested objects and arrays are supported."
---

## Phase 1 — Triage

MUST:
- Verify template data format is valid JSON
- Check variable names match between template and data (case-sensitive)
- Test rendering with sample data: `aws sesv2 test-render-email-template --template-name <name> --template-data '{"name":"Test","items":["a","b"]}'`
- Verify bulk email replacement data structure

SHOULD:
- Check for special characters in template data that need escaping
- Verify nested object access syntax: `{{address.city}}`
- Review Handlebars helper usage (if, each, unless)

MAY:
- Check template data size limits for bulk sending
- Verify encoding of non-ASCII characters in template data

## Phase 2 — Remediate

MUST:
- Ensure template data is valid JSON with correct variable names
- Match variable names exactly (case-sensitive) between template and data
- Provide default values for optional variables

SHOULD:
- Use test-render-email-template before production sends
- Implement data validation before passing to SES API

MAY:
- Create reusable template data schemas for consistency
- Implement template data generation in application code

## Common Issues

- symptoms: "Variables show as literal {{variable}} in email"
  diagnosis: "Variable name mismatch or template data not provided."
  resolution: "Verify variable names match exactly. Ensure template data is passed in API call."

- symptoms: "Bulk email fails for some recipients"
  diagnosis: "Per-recipient template data missing required variables."
  resolution: "Ensure each recipient's replacement data includes all required variables."

- symptoms: "Nested object values not rendering"
  diagnosis: "Incorrect dot notation or missing nested object in data."
  resolution: "Use correct syntax: {{address.city}}. Ensure nested object exists in data."

## Output Format

```yaml
root_cause: "personalization — <specific_cause>"
evidence:
  - type: template_data
    content: "<template data structure and variables>"
  - type: render_result
    content: "<test render output>"
severity: MEDIUM
mitigation:
  immediate: "Fix template data format or variable names"
  long_term: "Implement data validation and template testing"
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
