---
title: "A2 — Index Configuration Issues"
description: "Diagnose index settings, capacity, and attribute configuration"
status: active
severity: MEDIUM
triggers:
  - "index capacity exceeded"
  - "document attribute error"
  - "index update failed"
owner: devops-agent
objective: "Resolve index configuration issues"
context: "Index configuration includes edition, capacity units, document attributes, and user context settings."
---

## Phase 1 — Triage

MUST:
- Describe index: `aws kendra describe-index --id <index-id>`
- Check document count vs capacity limits
- Verify document attribute definitions
- Check capacity unit configuration

SHOULD:
- Review user context configuration
- Check encryption settings
- Verify index update history

MAY:
- Compare with recommended configurations
- Check for deprecated settings

## Phase 2 — Remediate

MUST:
- Add missing document attribute definitions
- Increase capacity units if needed
- Fix configuration errors

SHOULD:
- Plan capacity based on document growth
- Define all needed attributes before ingestion

## Common Issues

- symptoms: "ServiceQuotaExceededException on document ingestion"
  diagnosis: "Index document capacity exceeded."
  resolution: "Add capacity units or upgrade to Enterprise Edition."

## Output Format

```yaml
root_cause: "index_config — <specific_cause>"
evidence:
  - type: index_details
    content: "<index configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration"
  long_term: "Plan capacity proactively"
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
