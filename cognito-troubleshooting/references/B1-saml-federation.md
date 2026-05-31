---
title: "B1 — SAML Federation Issues"
description: "Diagnose SAML identity provider federation failures with Cognito"
status: active
severity: HIGH
triggers:
  - "SAML"
  - "SAML federation"
  - "IdP login failed"
  - "SAML assertion"
  - "federation error"
owner: devops-agent
objective: "Identify and fix SAML federation configuration issues between Cognito and external IdPs"
context: "SAML federation requires exact metadata exchange, attribute mapping, and endpoint configuration between Cognito and the IdP. Common failures include metadata mismatch, incorrect ACS URL, attribute mapping case sensitivity, and certificate expiry."
---

## Phase 1 — Triage

MUST:
- Check SAML provider configuration: `aws cognito-idp describe-identity-provider --user-pool-id <pool-id> --provider-name <provider-name>`
- Verify the user pool domain is configured: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.Domain'`
- Check app client OAuth configuration: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.{AllowedOAuthFlows:AllowedOAuthFlows,CallbackURLs:CallbackURLs,SupportedIdentityProviders:SupportedIdentityProviders}'`
- Verify the SAML provider is listed as a supported identity provider on the app client

SHOULD:
- Check attribute mapping configuration in the identity provider settings
- Verify the ACS URL matches: `https://<domain>.auth.<region>.amazoncognito.com/saml2/idpresponse`
- Check if the IdP metadata/certificate has expired
- Review CloudWatch logs for SAML assertion errors

MAY:
- Capture and decode the SAML assertion using browser developer tools
- Check if the IdP is sending the NameID in the expected format

## Phase 2 — Remediate

MUST:
- Fix attribute mapping to match exact IdP attribute names (case-sensitive)
- Update IdP metadata if certificate has expired
- Ensure the SAML provider is added to the app client's supported identity providers

SHOULD:
- Update the ACS URL in the IdP to match the Cognito endpoint
- Configure proper callback URLs in the app client

MAY:
- Enable CloudWatch logging for detailed SAML assertion debugging

## Common Issues

- symptoms: "Invalid SAML response or assertion"
  diagnosis: "SAML assertion format or signature is invalid."
  resolution: "Re-upload IdP metadata. Verify the IdP certificate matches."

- symptoms: "User authenticates at IdP but gets error at Cognito"
  diagnosis: "ACS URL mismatch or attribute mapping failure."
  resolution: "Verify ACS URL in IdP matches Cognito endpoint. Check attribute mapping case sensitivity."

- symptoms: "User attributes are empty after SAML sign-in"
  diagnosis: "Attribute mapping names don't match the SAML assertion attributes."
  resolution: "Decode the SAML assertion to find exact attribute names and update mapping."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix SAML attribute mapping | YELLOW | Configuration change; recoverable |
| Update IdP metadata | YELLOW | Metadata refresh; previous can be re-uploaded |
| Add SAML provider to app client | YELLOW | Configuration change; recoverable |
| Update ACS URL in IdP | YELLOW | External IdP change; reversible |

## Escalation Conditions

- Production user pool SAML provider configuration changes
- Identity pool role mapping changes
- Changes affecting enterprise SSO for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-identity-provider` | MEDIUM | Federation configuration details |
| `describe-user-pool-client` (OAuth) | MEDIUM | OAuth configuration |
| SAML assertion decode | HIGH | Contains user identity attributes |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER expose SAML signing certificates or private keys

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Attribute mapping change | Revert via `update-identity-provider` |
| IdP metadata update | Re-upload previous metadata |
| App client provider change | Remove provider via `update-user-pool-client` |

## Output Format

```yaml
root_cause: "saml_federation_failure — <specific_cause>"
evidence:
  - type: provider_config
    content: "<SAML provider configuration>"
  - type: attribute_mapping
    content: "<current attribute mapping>"
severity: HIGH
mitigation:
  immediate: "Fix SAML configuration or attribute mapping"
  long_term: "Implement SAML assertion logging and certificate expiry monitoring"
```

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
  - command: "describe-user-pool"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-users"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "admin-get-user"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling MFA for troubleshooting"
  - "NEVER expose user PII in investigation output"
  - "NEVER suggest removing password policies"
