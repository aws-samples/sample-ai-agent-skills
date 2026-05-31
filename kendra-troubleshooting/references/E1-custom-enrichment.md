---
title: "E1 — Custom Document Enrichment Issues"
description: "Diagnose custom document enrichment configuration and processing"
status: active
severity: MEDIUM
triggers:
  - "enrichment not applied"
  - "document metadata missing"
  - "CDE configuration error"
owner: devops-agent
objective: "Resolve custom document enrichment issues"
context: "CDE modifies document metadata during ingestion using inline configurations or Lambda functions."
---

## Phase 1 — Triage

MUST:
- Check data source CDE configuration
- Verify inline enrichment conditions and targets
- Check if Lambda hook is configured and accessible
- Review sync logs for enrichment errors

SHOULD:
- Verify target attribute is defined in index
- Check enrichment condition logic
- Test with specific documents

MAY:
- Check enrichment execution order
- Verify pre/post extraction hooks

## Phase 2 — Remediate

MUST:
- Fix enrichment configuration
- Define target attributes in index
- Fix Lambda function if used

SHOULD:
- Test enrichment with small document sets
- Document enrichment rules

## Common Issues

- symptoms: "Metadata not appearing on documents after sync"
  diagnosis: "Target attribute not defined in index or condition not matching."
  resolution: "Define attribute in index. Check condition logic. Re-sync."

## Output Format

```yaml
root_cause: "custom_enrichment — <specific_cause>"
evidence:
  - type: enrichment_config
    content: "<CDE configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix enrichment configuration"
  long_term: "Test enrichment rules systematically"
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
