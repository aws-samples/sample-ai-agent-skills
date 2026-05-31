---
title: "G2 — User Mapping Issues"
description: "Diagnose user-to-group mapping and permission problems"
status: active
severity: MEDIUM
triggers:
  - "user permissions wrong"
  - "group mapping error"
  - "user role incorrect"
owner: devops-agent
objective: "Resolve user mapping and permission issues"
context: "User roles and group memberships determine access to documents and features."
---

## Phase 1 — Triage

MUST:
- Check user assignments in Identity Center
- Verify group memberships
- Check user role (admin vs user)
- Verify ACL group mappings

SHOULD:
- Test with different user roles
- Check group sync from external IdP
- Verify persona assignments

## Phase 2 — Remediate

MUST:
- Fix user role assignments
- Correct group memberships
- Update ACL mappings

SHOULD:
- Document role requirements
- Test access with representative users

## Common Issues

- symptoms: "Admin user cannot manage application"
  diagnosis: "User assigned as regular user, not admin."
  resolution: "Update user role to admin in Identity Center assignment."

## Output Format

```yaml
root_cause: "user_mapping — <specific_cause>"
evidence:
  - type: user_config
    content: "<user mapping details>"
severity: MEDIUM
mitigation:
  immediate: "Fix user mappings"
  long_term: "Automate role management"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — user mapping controls data access permissions
- Service Disruption Risk: MEDIUM — mapping issues affect individual user access
- Compliance Impact: HIGH — user mapping enforces data access compliance

## Escalation Conditions
- User mapping fails causing access issues
- Incorrect mapping grants unauthorized data access
- Mapping synchronization fails
- User mapping conflicts between sources
- Bulk mapping changes affect many users

## Data Sensitivity
- User identity mappings — CONFIDENTIAL
- Data source access mappings — CONFIDENTIAL
- Group membership mappings — CONFIDENTIAL
- Mapping configurations — INTERNAL
- Never expose user mapping details externally

## Prohibited Actions
- Do NOT create mappings that grant unauthorized data access
- Do NOT bypass user mapping review processes
- Do NOT ignore mapping synchronization failures
- Do NOT share user mapping configurations externally
- Do NOT modify mappings without understanding access impact
- Do NOT disable user mapping validation

## Phase 3 — Rollback
- Restore previous user mapping configurations
- Revert mapping changes that caused access issues
- Re-synchronize user mappings
- Verify user access after mapping changes
- Document mapping issues and access impact
- Update user mapping procedures

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
