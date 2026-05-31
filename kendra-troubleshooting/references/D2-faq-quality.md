---
title: "D2 — FAQ Quality Issues"
description: "Diagnose FAQ answer quality and matching problems"
status: active
severity: LOW
triggers:
  - "FAQ not matching queries"
  - "wrong FAQ answer"
  - "FAQ not appearing in results"
owner: devops-agent
objective: "Improve FAQ matching and answer quality"
context: "FAQ matching depends on question similarity. Poor matches stem from vague questions or missing variations."
---

## Phase 1 — Triage

MUST:
- Test queries that should match FAQs
- Check FAQ question wording vs user queries
- Verify FAQ is in ACTIVE status
- Check result types in query response

SHOULD:
- Add question variations for better matching
- Check FAQ answer completeness
- Review FAQ ranking vs document results

MAY:
- Add URL references to FAQ answers
- Check for conflicting FAQ entries

## Phase 2 — Remediate

MUST:
- Improve FAQ question wording to match user queries
- Add question variations
- Remove duplicate or conflicting entries

SHOULD:
- Regularly update FAQs based on query analytics
- Test FAQ matching with common queries

## Common Issues

- symptoms: "FAQ exists but never appears in results"
  diagnosis: "FAQ question wording too different from user queries."
  resolution: "Add question variations. Use natural language phrasing."

## Output Format

```yaml
root_cause: "faq_quality — <specific_cause>"
evidence:
  - type: faq_matching
    content: "<FAQ match analysis>"
severity: LOW
mitigation:
  immediate: "Improve FAQ questions"
  long_term: "Regular FAQ maintenance"
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
