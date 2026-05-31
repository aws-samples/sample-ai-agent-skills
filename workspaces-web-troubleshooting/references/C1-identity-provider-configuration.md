---
title: "C1 — Identity Provider Configuration"
description: "Diagnose WorkSpaces Web identity provider setup issues"
status: active
severity: HIGH
triggers: ["IdP configuration error", "identity provider failed", "cannot add IdP", "SSO not working"]
owner: devops-agent
objective: "Resolve identity provider configuration issues for WorkSpaces Web portals"
context: "WorkSpaces Web requires a SAML 2.0 identity provider. The IdP must be configured with the portal's service provider metadata. Supported IdPs include Okta, Azure AD, Ping Identity, ADFS, and others with SAML 2.0 support."
---

## Phase 1 — Triage

MUST:
- List identity providers for the portal: `aws workspaces-web list-identity-providers --portal-arn <portal-arn>`
- Get IdP details: `aws workspaces-web get-identity-provider --portal-arn <portal-arn> --identity-provider-arn <idp-arn>`
- Verify portal has the service provider metadata URL for IdP configuration

SHOULD:
- Verify the IdP metadata XML is valid and contains the correct SSO URL and certificate
- Check that the SAML assertion attributes match WorkSpaces Web requirements
- Verify the IdP is reachable from the internet (not behind a firewall)

MAY:
- Review CloudTrail for CreateIdentityProvider or UpdateIdentityProvider errors
- Test SAML authentication flow using a browser developer tools network tab

## Phase 2 — Remediate

MUST:
- Create or update the identity provider with correct SAML metadata: `aws workspaces-web create-identity-provider --portal-arn <portal-arn> --identity-provider-name <name> --identity-provider-type SAML --identity-provider-details '{"MetadataURL":"<idp-metadata-url>"}'`
- Configure the IdP application with the portal's SP entity ID and ACS URL
- Ensure the SAML response includes the required NameID attribute

SHOULD:
- Map IdP user attributes to WorkSpaces Web expected attributes
- Configure IdP to sign both SAML response and assertion
- Test with a single user before enabling for all users

MAY:
- Configure group-based access control in the IdP
- Set up IdP-initiated SSO in addition to SP-initiated SSO

## Safety Ratings

```
safety_ratings:
  - "list-identity-providers: GREEN — read-only enumeration"
  - "get-identity-provider: GREEN — read-only IdP details"
  - "create-identity-provider: YELLOW — adds authentication configuration"
  - "update-identity-provider: RED — modifies authentication, may lock out users"
  - "delete-identity-provider: RED — removes authentication, locks out all users"
```

## Escalation Conditions

- IdP metadata URL is unreachable from AWS
- SAML assertion format does not match WorkSpaces Web requirements after multiple attempts
- IdP certificate is expired and renewal requires IdP admin access
- Organization requires a specific IdP that has SAML compatibility issues

## Data Sensitivity

- `aws workspaces-web list-identity-providers`: LOW — returns IdP metadata
- `aws workspaces-web get-identity-provider`: MEDIUM — reveals IdP configuration details
- IdP metadata XML: MEDIUM — contains SSO endpoints and certificates
- SAML assertions: HIGH — contain user identity and attribute information

## Prohibited Actions

- NEVER delete the identity provider from a production portal without a replacement configured
- NEVER share IdP metadata or certificates in unsecured channels
- NEVER configure SAML without encryption for production portals
- NEVER bypass SAML authentication for troubleshooting by suggesting alternative auth methods

## Phase 3 — Rollback

1. Restore previous IdP configuration if changes locked out users: `aws workspaces-web update-identity-provider --portal-arn <portal-arn> --identity-provider-arn <idp-arn> --identity-provider-details '{"MetadataURL":"<previous-url>"}'`
2. If the IdP was deleted, recreate it with the previous configuration
3. Verify at least one test user can authenticate after rollback
4. Notify affected users if authentication was disrupted
5. Document the IdP configuration change and rollback reason

## Common Issues

- symptoms: "Users see 'Invalid SAML response' after IdP authentication"
  diagnosis: "SAML assertion does not include required NameID or attribute mapping is incorrect."
  resolution: "Configure the IdP to include NameID in email format and map required attributes."

## Output Format

```yaml
root_cause: "identity_provider — <specific_cause>"
evidence:
  - type: idp_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IdP configuration"
  long_term: "Document IdP setup procedures and implement SAML monitoring"
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
