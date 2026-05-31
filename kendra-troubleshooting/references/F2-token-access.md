---
title: "F2 — Token-Based Access Control"
description: "Diagnose token-based user context and access control"
status: active
severity: MEDIUM
triggers:
  - "token validation error"
  - "JWT access control"
  - "user token not recognized"
owner: devops-agent
objective: "Resolve token-based access control issues"
context: "Token-based access uses JWT or JSON user context to determine document access."
---

## Phase 1 — Triage

MUST:
- Check user token configuration on index
- Verify token format (JWT, JSON, OpenID)
- Check token claims mapping
- Test with valid token

SHOULD:
- Verify token signing key configuration
- Check token expiration
- Verify group claim extraction

MAY:
- Decode token to inspect claims
- Check for token size limits

## Phase 2 — Remediate

MUST:
- Fix token configuration on index
- Correct claims mapping
- Update signing key if rotated

SHOULD:
- Document token format requirements
- Test with sample tokens

## Common Issues

- symptoms: "Query ignores user context from token"
  diagnosis: "Token configuration not set on index or claims mapping wrong."
  resolution: "Configure UserTokenConfiguration on index. Map correct claims."

## Output Format

```yaml
root_cause: "token_access — <specific_cause>"
evidence:
  - type: token_config
    content: "<token configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix token configuration"
  long_term: "Automate token validation testing"
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
