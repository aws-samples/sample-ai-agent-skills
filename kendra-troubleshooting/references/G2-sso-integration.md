---
title: "G2 — SSO Integration Issues"
description: "Diagnose IAM Identity Center integration with Kendra"
status: active
severity: MEDIUM
triggers:
  - "SSO login failed"
  - "user not found in experience"
  - "identity center error"
owner: devops-agent
objective: "Resolve SSO integration issues with Kendra"
context: "Kendra Experience Builder uses IAM Identity Center for authentication."
---

## Phase 1 — Triage

MUST:
- Verify IAM Identity Center is enabled
- Check user provisioning in Identity Center
- Verify user/group assignment to experience
- Check experience authentication configuration

SHOULD:
- Test SSO login flow
- Verify group membership
- Check for Identity Center sync issues

MAY:
- Check SAML/SCIM configuration if using external IdP
- Review Identity Center audit logs

## Phase 2 — Remediate

MUST:
- Provision users in IAM Identity Center
- Assign users to Kendra experience
- Fix authentication configuration

SHOULD:
- Sync groups from external IdP
- Test with multiple user roles

## Common Issues

- symptoms: "User can access experience but sees no results"
  diagnosis: "User not mapped to correct groups for ACL access."
  resolution: "Check group membership. Verify ACL group mappings."

## Output Format

```yaml
root_cause: "sso_integration — <specific_cause>"
evidence:
  - type: sso_config
    content: "<SSO configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix SSO configuration"
  long_term: "Automate user provisioning"
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
