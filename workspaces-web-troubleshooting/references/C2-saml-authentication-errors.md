---
title: "C2 — SAML Authentication Errors"
description: "Diagnose WorkSpaces Web SAML authentication failures"
status: active
severity: HIGH
triggers: ["SAML error", "authentication failed", "login failed", "SSO error", "invalid assertion"]
owner: devops-agent
objective: "Resolve SAML authentication errors preventing user access to WorkSpaces Web"
context: "SAML authentication involves the portal (SP), the identity provider (IdP), and the user's browser. Errors can occur at any point in the SAML flow — IdP authentication, assertion generation, or SP validation."
---

## Phase 1 — Triage

MUST:
- Verify IdP configuration: `aws workspaces-web get-identity-provider --portal-arn <portal-arn> --identity-provider-arn <idp-arn>`
- Check if the error affects all users or specific users
- Capture the SAML error message from the browser (URL parameters or error page)

SHOULD:
- Check IdP logs for authentication failures or assertion generation errors
- Verify the IdP certificate has not expired
- Confirm the ACS URL in the IdP matches the portal's expected ACS URL

MAY:
- Use a SAML debugging browser extension to inspect the SAML response
- Check CloudTrail for authentication-related events
- Verify clock synchronization between IdP and AWS (SAML has time-based validity)

## Phase 2 — Remediate

MUST:
- Fix SAML assertion format to match WorkSpaces Web requirements
- Update expired IdP certificates in both the IdP and WorkSpaces Web configuration
- Ensure the ACS URL and Entity ID match between IdP and portal

SHOULD:
- Verify NameID format is set to email address format
- Ensure SAML response validity period is reasonable (not too short)
- Configure the IdP to sign both the response and the assertion

MAY:
- Enable SAML debugging in the IdP for detailed error logging
- Configure attribute mapping for additional user attributes
- Set up IdP-initiated SSO as an alternative access method

## Safety Ratings

```
safety_ratings:
  - "get-identity-provider: GREEN — read-only IdP configuration"
  - "update-identity-provider: RED — modifies authentication configuration"
  - "IdP certificate rotation: RED — affects all users if done incorrectly"
  - "SAML debugging tools: GREEN — read-only inspection of SAML flow"
```

## Escalation Conditions

- All users locked out due to SAML configuration error
- IdP certificate expired and IdP admin is unavailable
- SAML assertion format incompatibility between IdP and WorkSpaces Web
- Clock skew between IdP and AWS exceeds SAML validity window

## Data Sensitivity

- `aws workspaces-web get-identity-provider`: MEDIUM — reveals IdP configuration
- SAML assertions: HIGH — contain user identity, email, and group membership
- IdP logs: HIGH — contain authentication attempts and user identifiers
- Browser SAML debug output: HIGH — contains full SAML response with user data

## Prohibited Actions

- NEVER share SAML assertions or IdP certificates in unsecured channels
- NEVER suggest disabling SAML validation or signature verification
- NEVER suggest alternative authentication methods — SAML 2.0 is the only option
- NEVER access or log user credentials during SAML troubleshooting

## Phase 3 — Rollback

1. Restore previous IdP certificate if a rotation caused the issue
2. Revert ACS URL or Entity ID changes in the IdP if they broke authentication
3. Restore previous attribute mapping if changes caused assertion validation failures
4. Verify authentication works for a test user after rollback
5. Notify affected users when authentication is restored

## Common Issues

- symptoms: "Error: SAML response expired"
  diagnosis: "Clock skew between IdP server and AWS exceeds the SAML assertion validity window."
  resolution: "Synchronize the IdP server clock with NTP. Increase SAML assertion validity period in IdP settings."

## Output Format

```yaml
root_cause: "saml_authentication — <specific_cause>"
evidence:
  - type: auth_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix SAML authentication issue"
  long_term: "Implement SAML monitoring and certificate rotation procedures"
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
