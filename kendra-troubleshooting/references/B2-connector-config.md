---
title: "B2 — Data Source Connector Configuration"
description: "Diagnose connector-specific configuration issues"
status: active
severity: MEDIUM
triggers:
  - "connector configuration error"
  - "data source creation failed"
  - "authentication error on connector"
owner: devops-agent
objective: "Resolve data source connector configuration issues"
context: "Each connector type (S3, SharePoint, Salesforce, etc.) has specific configuration requirements."
---

## Phase 1 — Triage

MUST:
- Check data source type and configuration
- Verify connector credentials in Secrets Manager
- Check endpoint URL accessibility
- Verify IAM role permissions for the connector type

SHOULD:
- Check connector-specific documentation
- Verify crawl scope configuration
- Test credentials independently

MAY:
- Check for connector version updates
- Verify SSL/TLS certificate validity for endpoints

## Phase 2 — Remediate

MUST:
- Fix connector credentials
- Correct endpoint URLs
- Update IAM permissions

SHOULD:
- Document connector configurations
- Test with minimal crawl scope first

## Common Issues

- symptoms: "SharePoint connector fails with 401"
  diagnosis: "SharePoint credentials expired or insufficient permissions."
  resolution: "Update credentials in Secrets Manager. Verify SharePoint app permissions."

## Output Format

```yaml
root_cause: "connector_config — <specific_cause>"
evidence:
  - type: connector_details
    content: "<connector configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix connector configuration"
  long_term: "Monitor credential expiration"
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
