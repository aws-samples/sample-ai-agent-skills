---
title: "G1 — IAM Identity Center Integration"
description: "Diagnose IAM Identity Center setup and integration issues"
status: active
severity: HIGH
triggers:
  - "SSO not working"
  - "Identity Center error"
  - "user cannot login"
owner: devops-agent
objective: "Resolve IAM Identity Center integration issues"
context: "Q Business requires IAM Identity Center for all user authentication."
---

## Phase 1 — Triage

MUST:
- Verify Identity Center is enabled: `aws sso-admin list-instances`
- Check Identity Center instance region matches Q Business
- Verify user provisioning
- Check application assignment

SHOULD:
- Test SSO login flow
- Verify external IdP configuration if used
- Check SCIM provisioning status

## Phase 2 — Remediate

MUST:
- Enable Identity Center if not configured
- Provision users
- Assign users to Q Business application

SHOULD:
- Configure group-based access
- Test with multiple user types

## Common Issues

- symptoms: "Users cannot log in to Q Business"
  diagnosis: "Users not provisioned in Identity Center."
  resolution: "Provision users in Identity Center. Assign to Q Business application."

## Output Format

```yaml
root_cause: "identity_center — <specific_cause>"
evidence:
  - type: sso_config
    content: "<Identity Center configuration>"
severity: HIGH
mitigation:
  immediate: "Fix Identity Center configuration"
  long_term: "Automate user provisioning"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — Identity Center controls all user authentication
- Service Disruption Risk: HIGH — Identity Center failures prevent all user access
- Compliance Impact: HIGH — identity management directly affects security compliance

## Escalation Conditions
- Identity Center integration completely fails
- User authentication breaks for all users
- Identity Center configuration changes affect access
- User synchronization fails
- Unauthorized access detected

## Data Sensitivity
- Identity Center configurations — CONFIDENTIAL
- User directory information — CONFIDENTIAL
- Authentication tokens — RESTRICTED
- Group membership details — CONFIDENTIAL
- Never expose Identity Center credentials

## Prohibited Actions
- Do NOT bypass Identity Center authentication
- Do NOT modify Identity Center settings without testing
- Do NOT grant admin access without authorization
- Do NOT share Identity Center credentials
- Do NOT disable Identity Center integration
- Do NOT ignore authentication failures

## Phase 3 — Rollback
- Restore previous Identity Center configurations
- Re-establish user synchronization
- Revert group membership changes
- Verify user authentication after changes
- Document Identity Center issues
- Update identity management procedures

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
