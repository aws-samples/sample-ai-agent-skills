---
title: "F1 — ACL Configuration Issues"
description: "Diagnose access control list configuration and filtering"
status: active
severity: HIGH
triggers:
  - "users see no results"
  - "ACL filtering wrong"
  - "access control error"
owner: devops-agent
objective: "Resolve ACL configuration issues"
context: "ACLs control document visibility per user. Misconfigured ACLs cause missing results."
---

## Phase 1 — Triage

MUST:
- Check data source ACL configuration
- Verify user context is included in queries
- Test query with and without user context
- Check user/group mappings

SHOULD:
- Verify ACL source (data source native vs custom)
- Check group resolution configuration
- Test with admin user context

MAY:
- Review ACL sync logs
- Check for ACL propagation delays

## Phase 2 — Remediate

MUST:
- Fix user context in query calls
- Correct ACL mappings
- Re-sync if ACL configuration changed

SHOULD:
- Test ACLs with representative users
- Document ACL configuration

## Common Issues

- symptoms: "Query returns 0 results for specific users"
  diagnosis: "User context missing or ACL denying access."
  resolution: "Include UserContext in query. Verify user has access to documents."

## Output Format

```yaml
root_cause: "acl_config — <specific_cause>"
evidence:
  - type: acl_details
    content: "<ACL configuration>"
severity: HIGH
mitigation:
  immediate: "Fix ACL configuration"
  long_term: "Test ACLs with user personas"
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
