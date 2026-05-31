---
title: "A2 — Admin Console Access"
description: "Diagnose Wickr admin console access issues"
status: active
severity: HIGH
triggers: ["cannot access admin console", "admin login failed", "console access denied"]
owner: devops-agent
objective: "Resolve admin console authentication and access issues"
context: "Admin console requires IAM Identity Center (SSO) or SAML federation. Direct IAM user login is not supported."
---

## Phase 1 — Triage

MUST:
- Verify IAM Identity Center is configured: `aws sso-admin list-instances`
- Check if user is assigned to the Wickr application in IAM Identity Center
- Review CloudTrail for authentication events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 10`

SHOULD:
- Verify SAML configuration if using external IdP
- Check if the user's IAM Identity Center session is active
- Verify the Wickr network exists and is active: `aws wickr get-network --network-id <id>`

MAY:
- Test with a different admin user to isolate user-specific issues
- Check browser console for authentication errors

## Phase 2 — Remediate

MUST:
- Configure IAM Identity Center application assignment for the admin user
- Verify SAML metadata is current and not expired
- Ensure the admin user has the correct Wickr admin role

SHOULD:
- Update SAML certificate if expired
- Clear browser cache and cookies for the admin console URL
- Verify network connectivity to the admin console endpoint

MAY:
- Configure additional admin users for redundancy
- Set up MFA for admin console access

## Safety Ratings

```
safety_ratings:
  - "sso-admin list-instances: GREEN — read-only SSO enumeration"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "wickr get-network: GREEN — read-only network details"
  - "SSO application assignment: YELLOW — grants admin access"
```

## Escalation Conditions

- All admin users locked out of the console simultaneously
- SAML federation completely broken with no fallback access
- IAM Identity Center instance is misconfigured or deleted
- Admin console returns 500 errors consistently

## Data Sensitivity

- `aws sso-admin list-instances`: LOW — returns SSO instance metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals authentication attempts and principal identities
- Admin user assignments: MEDIUM — reveals who has administrative access

## Prohibited Actions

- NEVER bypass SSO/SAML authentication for admin console access
- NEVER share admin credentials or session tokens
- NEVER remove all admin users from the Wickr network
- NEVER modify IAM Identity Center configuration without understanding impact on other applications

## Phase 3 — Rollback

1. If SSO application assignment was modified, restore original user assignments
2. If SAML configuration was changed, restore previous metadata document
3. Verify all legitimate admin users can access the console after changes
4. Confirm no unauthorized users gained admin access during troubleshooting
5. Document authentication changes and notify the security team

## Common Issues

- symptoms: "Admin console shows 'Access Denied' after SSO login"
  diagnosis: "User is not assigned to the Wickr application in IAM Identity Center."
  resolution: "Assign the user to the Wickr application with admin permissions in IAM Identity Center."

- symptoms: "SAML authentication fails with certificate error"
  diagnosis: "SAML signing certificate has expired."
  resolution: "Update the SAML metadata with a current certificate from the identity provider."

## Output Format

```yaml
root_cause: "admin_console_access — <specific_cause>"
evidence:
  - type: authentication_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore admin console access"
  long_term: "Configure redundant admin access and certificate rotation"
```

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
