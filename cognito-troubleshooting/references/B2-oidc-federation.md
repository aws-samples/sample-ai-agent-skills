---
title: "B2 — OIDC Federation Issues"
description: "Diagnose OpenID Connect identity provider federation failures"
status: active
severity: HIGH
triggers:
  - "OIDC"
  - "OpenID Connect"
  - "social login"
  - "Google login"
  - "Facebook login"
owner: devops-agent
objective: "Identify and fix OIDC federation configuration issues"
context: "OIDC federation requires correct client ID, client secret, issuer URL, and attribute mapping. Common failures include incorrect issuer URL, mismatched redirect URIs, and scope configuration issues."
---

## Phase 1 — Triage

MUST:
- Check OIDC provider configuration: `aws cognito-idp describe-identity-provider --user-pool-id <pool-id> --provider-name <provider-name>`
- Verify app client OAuth settings: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.{AllowedOAuthFlows:AllowedOAuthFlows,AllowedOAuthScopes:AllowedOAuthScopes,CallbackURLs:CallbackURLs}'`
- Check that the OIDC provider is in the app client's supported identity providers
- Verify the user pool domain exists: `aws cognito-idp describe-user-pool-domain --domain <domain>`

SHOULD:
- Verify the OIDC issuer URL is accessible and returns valid discovery document
- Check that the redirect URI in the OIDC provider matches Cognito's callback
- Review attribute mapping for the OIDC provider

MAY:
- Test the OIDC discovery endpoint directly
- Check if the OIDC provider's client secret has been rotated

## Phase 2 — Remediate

MUST:
- Fix issuer URL, client ID, or client secret if incorrect
- Add the OIDC provider to the app client's supported identity providers
- Configure correct callback URLs

SHOULD:
- Update attribute mapping to match OIDC claims
- Ensure required OAuth scopes are configured (openid, email, profile)

MAY:
- Implement token validation on the application side

## Common Issues

- symptoms: "Error during OIDC redirect — invalid_client"
  diagnosis: "Client ID or secret mismatch between Cognito and the OIDC provider."
  resolution: "Verify client ID and secret match the OIDC provider's configuration."

- symptoms: "Redirect URI mismatch error"
  diagnosis: "The callback URL in the OIDC provider doesn't match Cognito's endpoint."
  resolution: "Set redirect URI to https://<domain>.auth.<region>.amazoncognito.com/oauth2/idpresponse"

- symptoms: "User attributes missing after OIDC sign-in"
  diagnosis: "OIDC scopes don't include the required claims or attribute mapping is wrong."
  resolution: "Add openid, email, profile scopes. Update attribute mapping."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix OIDC issuer/client config | YELLOW | Configuration change; recoverable |
| Add OIDC provider to app client | YELLOW | Configuration change; recoverable |
| Update attribute mapping | YELLOW | Mapping change; reversible |

## Escalation Conditions

- Production user pool OIDC provider configuration changes
- Identity pool role mapping changes
- Changes affecting social login for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-identity-provider` | HIGH | Contains client ID and potentially secret |
| `describe-user-pool-client` (OAuth) | MEDIUM | OAuth configuration |
| `describe-user-pool-domain` | LOW | Domain configuration |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER expose OIDC client secrets in logs or output

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| OIDC provider config change | Revert via `update-identity-provider` |
| App client OAuth change | Revert via `update-user-pool-client` |
| Attribute mapping change | Revert via `update-identity-provider` |

## Output Format

```yaml
root_cause: "oidc_federation_failure — <specific_cause>"
evidence:
  - type: provider_config
    content: "<OIDC provider settings>"
  - type: oauth_config
    content: "<app client OAuth configuration>"
severity: HIGH
mitigation:
  immediate: "Fix OIDC provider configuration"
  long_term: "Monitor OIDC provider certificate and secret rotation"
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
