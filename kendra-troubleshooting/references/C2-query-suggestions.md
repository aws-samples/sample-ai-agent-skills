---
title: "C2 — Query Suggestions Issues"
description: "Diagnose query suggestion generation and quality"
status: active
severity: LOW
triggers:
  - "no query suggestions"
  - "suggestions not appearing"
  - "wrong suggestions"
owner: devops-agent
objective: "Resolve query suggestion issues"
context: "Query suggestions are generated from query history. Minimum query volume required."
---

## Phase 1 — Triage

MUST:
- Check suggestions config: `aws kendra describe-query-suggestions-config --index-id <index-id>`
- Verify suggestions are enabled
- Check query history volume
- Verify minimum unique queries threshold

SHOULD:
- Check suggestion block list
- Review suggestion mode (ENABLED/LEARN_ONLY)

MAY:
- Check query log retention settings
- Review suggestion quality metrics

## Phase 2 — Remediate

MUST:
- Enable suggestions if disabled
- Wait for sufficient query history to accumulate
- Clear block list if blocking valid suggestions

SHOULD:
- Configure suggestion settings appropriately
- Monitor suggestion quality

## Common Issues

- symptoms: "GetQuerySuggestions returns empty"
  diagnosis: "Insufficient query history."
  resolution: "Wait for more queries. Minimum ~1000 unique queries needed."

## Output Format

```yaml
root_cause: "query_suggestions — <specific_cause>"
evidence:
  - type: suggestion_config
    content: "<suggestion settings>"
severity: LOW
mitigation:
  immediate: "Enable and configure suggestions"
  long_term: "Monitor suggestion quality"
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
