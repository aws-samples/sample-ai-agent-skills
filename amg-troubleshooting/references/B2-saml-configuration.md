---
title: "B2 — SAML Configuration Issues"
description: "Diagnose SAML authentication configuration problems with Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "SAML error"
  - "SAML login failed"
  - "SAML configuration"
  - "identity provider"
  - "IdP metadata"
owner: devops-agent
objective: "Identify and resolve SAML configuration issues preventing authentication to an AMG workspace"
context: "AMG supports SAML 2.0 for authentication with external identity providers (Okta, Azure AD, Ping Identity, etc.). SAML configuration requires correct metadata exchange, attribute mapping, and assertion consumer service URL. Misconfigurations in any of these cause login failures."
---

## Phase 1 — Triage

MUST:
- Check workspace authentication configuration: `aws grafana describe-workspace-authentication --workspace-id <workspace-id>`
- Verify SAML is the configured authentication mode
- Check the SAML configuration details including IdP metadata URL or XML
- Verify the assertion consumer service (ACS) URL matches the workspace endpoint

SHOULD:
- Check SAML attribute mappings (role, name, email, login, org)
- Verify the IdP metadata is accessible and valid
- Review CloudTrail for SAML-related events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=grafana.amazonaws.com --max-results 20`

MAY:
- Use a SAML debugging tool to inspect the SAML assertion
- Check IdP-side configuration for the Grafana application

## Phase 2 — Remediate

MUST:
- Ensure the IdP metadata URL is accessible or the metadata XML is valid
- Configure correct attribute mappings for role, email, and login attributes
- Verify the ACS URL in the IdP matches: `https://<workspace-endpoint>/saml/acs`

SHOULD:
- Configure role attribute mapping to automatically assign Grafana roles (ADMIN, EDITOR, VIEWER)
- Set allowed organizations if using org attribute mapping
- Test with a single user before rolling out to all users

MAY:
- Configure assertion encryption if required by security policy
- Set up SAML group-to-role mapping for automated role assignment

## Common Issues

- symptoms: "SAML login fails with 'Invalid SAML response' error"
  diagnosis: "IdP metadata is incorrect, expired, or the ACS URL does not match."
  resolution: "Update the SAML configuration with correct IdP metadata and verify ACS URL matches workspace endpoint."

- symptoms: "User authenticates but gets wrong Grafana role"
  diagnosis: "SAML role attribute mapping is incorrect or the IdP is not sending the role attribute."
  resolution: "Configure the role assertion attribute name and ensure the IdP sends ADMIN, EDITOR, or VIEWER values."

- symptoms: "SAML login works but user email/name is missing"
  diagnosis: "Attribute mappings for email and name do not match the IdP's SAML assertion attribute names."
  resolution: "Update SAML attribute mappings to match the attribute names in the IdP's SAML assertion."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace-authentication: GREEN — read-only SAML config check"
  - "update SAML configuration: YELLOW — changes authentication flow for all users"
  - "update role attribute mapping: YELLOW — changes user role assignments"
  - "configure assertion encryption: YELLOW — changes security posture of auth flow"
```

## Escalation Conditions

- All users locked out due to SAML misconfiguration on a production workspace
- IdP metadata URL is inaccessible and no cached metadata is available
- SAML configuration changes require coordination with external IdP administrators
- Role mapping changes affect user access across the organization

## Data Sensitivity

- SAML configuration details: HIGH — reveals authentication flow and IdP integration
- IdP metadata: MEDIUM — contains IdP certificates and endpoints
- Role attribute mapping: MEDIUM — reveals how roles are assigned to users

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER change SAML configuration on a production workspace without a tested rollback plan
- NEVER share IdP metadata or SAML assertions in logs or support tickets without redacting sensitive fields

## Phase 3 — Rollback

1. If SAML configuration was changed and broke authentication, restore previous IdP metadata and attribute mappings
2. Use the AWS CLI to update SAML configuration back to the working state
3. If the workspace is completely inaccessible, switch to SSO authentication temporarily as a fallback
4. Coordinate with IdP administrators to verify the ACS URL and attribute mappings
5. Test authentication with a single user before restoring access for all users

## Output Format

```yaml
root_cause: "saml_configuration — <specific_cause>"
evidence:
  - type: saml_config
    content: "<SAML configuration details>"
  - type: attribute_mapping
    content: "<role, email, login attribute mappings>"
  - type: idp_metadata
    content: "<IdP metadata status>"
severity: HIGH
mitigation:
  immediate: "Fix SAML configuration to match IdP settings"
  long_term: "Document SAML attribute mappings and test after any IdP changes"
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
