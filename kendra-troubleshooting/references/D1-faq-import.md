---
title: "D1 — FAQ Import Failures"
description: "Diagnose FAQ file import and processing errors"
status: active
severity: MEDIUM
triggers:
  - "FAQ import failed"
  - "FAQ not available"
  - "FAQ format error"
owner: devops-agent
objective: "Resolve FAQ import failures"
context: "FAQs use CSV or JSON format. Format errors cause import failures."
---

## Phase 1 — Triage

MUST:
- List FAQs: `aws kendra list-faqs --index-id <index-id>`
- Check FAQ status (ACTIVE, CREATING, FAILED)
- Verify S3 path and permissions for FAQ file
- Check FAQ file format (CSV headers or JSON schema)

SHOULD:
- Validate CSV encoding (UTF-8 required)
- Check for special characters in FAQ content
- Verify FAQ file size limits

MAY:
- Test with a small FAQ file first
- Check for duplicate questions

## Phase 2 — Remediate

MUST:
- Fix FAQ file format (CSV: question,answer or question,answer,url)
- Ensure UTF-8 encoding
- Fix S3 permissions

SHOULD:
- Validate FAQ files before import
- Use JSON format for complex FAQs

## Common Issues

- symptoms: "FAQ stuck in CREATING then FAILED"
  diagnosis: "CSV format error or S3 access denied."
  resolution: "Verify CSV headers match expected format. Check S3 bucket policy."

## Output Format

```yaml
root_cause: "faq_import — <specific_cause>"
evidence:
  - type: faq_status
    content: "<FAQ import status>"
severity: MEDIUM
mitigation:
  immediate: "Fix FAQ file and re-import"
  long_term: "Validate FAQ files in CI/CD"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete indexes that serve production search applications
- Never modify data source connectors during active sync operations
- Never remove FAQ entries without confirming impact on search quality
- Never alter access control configurations without security review
- Never delete experience builder applications used by end users

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous data source connector configuration if sync operations fail
- Revert index relevance tuning settings if search quality degrades
- Re-enable previous ACL configurations if access control breaks

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
