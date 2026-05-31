---
title: "C3 — Query Filtering Issues"
description: "Diagnose attribute filtering and faceted search problems"
status: active
severity: LOW
triggers:
  - "filter not working"
  - "facets empty"
  - "attribute filter error"
owner: devops-agent
objective: "Resolve query filtering and facet issues"
context: "Query filters use document attributes. Attributes must be defined and facetable."
---

## Phase 1 — Triage

MUST:
- Verify attribute is defined in index: `aws kendra describe-index --id <index-id>`
- Check attribute is marked as searchable/facetable
- Test query with and without filters
- Verify attribute values exist in indexed documents

SHOULD:
- Check attribute type matches filter operation
- Verify attribute mapping in data source configuration

MAY:
- Check for attribute name case sensitivity
- Review facet configuration

## Phase 2 — Remediate

MUST:
- Define missing attributes in index configuration
- Enable facetable/searchable flags
- Re-sync data sources after attribute changes

SHOULD:
- Document available filter attributes
- Test filters with known documents

## Common Issues

- symptoms: "AttributeFilter returns no results"
  diagnosis: "Attribute not defined or not populated in documents."
  resolution: "Define attribute in index. Verify data source maps the attribute. Re-sync."

## Output Format

```yaml
root_cause: "query_filtering — <specific_cause>"
evidence:
  - type: attribute_config
    content: "<attribute definitions>"
severity: LOW
mitigation:
  immediate: "Fix attribute configuration"
  long_term: "Document filter capabilities"
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
